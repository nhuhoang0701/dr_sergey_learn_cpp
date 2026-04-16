# Test embedded bootloaders and firmware update paths

**Category:** Testing in Practice

---

## Topic Overview

**Bootloaders** are the most critical embedded code — a bug can permanently brick a device. Testing bootloaders requires simulating the entire update process: image download, verification, flash programming, rollback on failure, and multi-slot management. Host-based testing with simulated flash catches most logic bugs.

### Bootloader Test Coverage Matrix

| Scenario | Risk | Test Method |
| --- | --- | --- |
| Valid update | Basic functionality | Host simulation |
| Corrupted image | Bricked device | Inject bad CRC/signature |
| Power loss during write | Incomplete flash | Simulate partial write |
| Rollback to previous | Recovery path | Force update failure |
| Version downgrade | Security bypass | Version check tests |
| Full flash | No space for backup | Boundary tests |
| Signature verification | Unauthorized code | Inject bad signatures |

---

## Self-Assessment

### Q1: Design testable bootloader architecture with simulated flash

**Answer:**

```cpp

#include <cstdint>
#include <vector>
#include <span>
#include <optional>
#include <array>
#include <cstring>
#include <numeric>

// === Flash abstraction ===
class IFlashDriver {
public:
    virtual ~IFlashDriver() = default;
    virtual bool read(uint32_t address, std::span<uint8_t> buffer) = 0;
    virtual bool write(uint32_t address, std::span<const uint8_t> data) = 0;
    virtual bool erase_sector(uint32_t address) = 0;
    virtual uint32_t sector_size() const = 0;
    virtual uint32_t total_size() const = 0;
};

// === Flash simulator for testing ===
class FlashSimulator : public IFlashDriver {
public:
    static constexpr uint32_t SECTOR = 4096;
    static constexpr uint32_t SIZE = 256 * 1024;  // 256KB

    FlashSimulator() : mem_(SIZE, 0xFF) {}

    bool read(uint32_t addr, std::span<uint8_t> buf) override {
        if (addr + buf.size() > SIZE) return false;
        std::memcpy(buf.data(), &mem_[addr], buf.size());
        return true;
    }

    bool write(uint32_t addr, std::span<const uint8_t> data) override {
        if (fail_on_next_write_) { fail_on_next_write_ = false; return false; }
        if (addr + data.size() > SIZE) return false;
        for (size_t i = 0; i < data.size(); ++i)
            mem_[addr + i] &= data[i];  // Flash physics
        return true;
    }

    bool erase_sector(uint32_t addr) override {
        if (addr % SECTOR != 0 || addr >= SIZE) return false;
        std::fill_n(mem_.begin() + addr, SECTOR, 0xFF);
        return true;
    }

    uint32_t sector_size() const override { return SECTOR; }
    uint32_t total_size() const override { return SIZE; }

    // Test helpers
    void inject_write_failure() { fail_on_next_write_ = true; }
    std::span<const uint8_t> raw() const { return mem_; }

private:
    std::vector<uint8_t> mem_;
    bool fail_on_next_write_ = false;
};

// === Firmware image header ===
struct FirmwareHeader {
    uint32_t magic = 0x464D5752;  // "FMWR"
    uint32_t version;
    uint32_t size;       // Payload size (excluding header)
    uint32_t crc32;      // CRC of payload
    uint8_t  reserved[16];
};
static_assert(sizeof(FirmwareHeader) == 32);

// === Bootloader logic (under test) ===
class Bootloader {
public:
    static constexpr uint32_t SLOT_A_ADDR = 0x00000;
    static constexpr uint32_t SLOT_B_ADDR = 0x20000;
    static constexpr uint32_t MAX_FW_SIZE = 0x1F000;

    enum class UpdateResult {
        Success, BadMagic, BadCrc, TooLarge, WriteError,
        VersionDowngrade, FlashError
    };

    explicit Bootloader(IFlashDriver& flash) : flash_(flash) {}

    UpdateResult apply_update(std::span<const uint8_t> image,
                               uint32_t target_slot) {
        if (image.size() < sizeof(FirmwareHeader))
            return UpdateResult::BadMagic;

        FirmwareHeader header;
        std::memcpy(&header, image.data(), sizeof(header));

        if (header.magic != 0x464D5752)
            return UpdateResult::BadMagic;
        if (header.size > MAX_FW_SIZE)
            return UpdateResult::TooLarge;
        if (header.size + sizeof(FirmwareHeader) > image.size())
            return UpdateResult::BadMagic;

        // Verify CRC of payload
        auto payload = image.subspan(sizeof(FirmwareHeader), header.size);
        if (compute_crc32(payload) != header.crc32)
            return UpdateResult::BadCrc;

        // Check version (no downgrades)
        auto current = read_current_version(target_slot);
        if (current && header.version <= *current)
            return UpdateResult::VersionDowngrade;

        // Erase target slot
        for (uint32_t addr = target_slot;
             addr < target_slot + sizeof(FirmwareHeader) + header.size;
             addr += flash_.sector_size()) {
            if (!flash_.erase_sector(addr))
                return UpdateResult::FlashError;
        }

        // Write image
        if (!flash_.write(target_slot, image.subspan(0, sizeof(FirmwareHeader) + header.size)))
            return UpdateResult::WriteError;

        // Verify written data
        std::vector<uint8_t> verify(sizeof(FirmwareHeader) + header.size);
        flash_.read(target_slot, verify);
        if (std::memcmp(verify.data(), image.data(), verify.size()) != 0)
            return UpdateResult::WriteError;

        return UpdateResult::Success;
    }

private:
    std::optional<uint32_t> read_current_version(uint32_t slot) {
        FirmwareHeader header;
        std::array<uint8_t, sizeof(FirmwareHeader)> buf{};
        flash_.read(slot, buf);
        std::memcpy(&header, buf.data(), sizeof(header));
        if (header.magic != 0x464D5752) return std::nullopt;
        return header.version;
    }

    uint32_t compute_crc32(std::span<const uint8_t> data) {
        uint32_t crc = 0xFFFFFFFF;
        for (uint8_t byte : data) {
            crc ^= byte;
            for (int i = 0; i < 8; ++i)
                crc = (crc >> 1) ^ (0xEDB88320 & -(crc & 1));
        }
        return ~crc;
    }

    IFlashDriver& flash_;
};

```

### Q2: Test all bootloader update scenarios

**Answer:**

```cpp

#include <gtest/gtest.h>

class BootloaderTest : public ::testing::Test {
protected:
    // Helper: create a valid firmware image
    std::vector<uint8_t> make_image(uint32_t version,
                                     const std::vector<uint8_t>& payload) {
        FirmwareHeader hdr{};
        hdr.magic = 0x464D5752;
        hdr.version = version;
        hdr.size = payload.size();
        hdr.crc32 = compute_crc32(payload);

        std::vector<uint8_t> image(sizeof(hdr) + payload.size());
        std::memcpy(image.data(), &hdr, sizeof(hdr));
        std::memcpy(image.data() + sizeof(hdr), payload.data(), payload.size());
        return image;
    }

    uint32_t compute_crc32(const std::vector<uint8_t>& data) {
        uint32_t crc = 0xFFFFFFFF;
        for (uint8_t b : data) {
            crc ^= b;
            for (int i = 0; i < 8; ++i)
                crc = (crc >> 1) ^ (0xEDB88320 & -(crc & 1));
        }
        return ~crc;
    }

    FlashSimulator flash_;
    Bootloader bootloader_{flash_};
};

TEST_F(BootloaderTest, ValidUpdateSucceeds) {
    auto image = make_image(1, {0x01, 0x02, 0x03, 0x04});
    auto result = bootloader_.apply_update(image, Bootloader::SLOT_A_ADDR);
    EXPECT_EQ(result, Bootloader::UpdateResult::Success);
}

TEST_F(BootloaderTest, BadMagicRejected) {
    auto image = make_image(1, {0x01});
    image[0] = 0xFF;  // Corrupt magic
    EXPECT_EQ(bootloader_.apply_update(image, Bootloader::SLOT_A_ADDR),
              Bootloader::UpdateResult::BadMagic);
}

TEST_F(BootloaderTest, BadCrcRejected) {
    auto image = make_image(1, {0x01, 0x02, 0x03});
    image.back() ^= 0xFF;  // Corrupt payload
    EXPECT_EQ(bootloader_.apply_update(image, Bootloader::SLOT_A_ADDR),
              Bootloader::UpdateResult::BadCrc);
}

TEST_F(BootloaderTest, VersionDowngradeRejected) {
    auto v2 = make_image(2, {0xAA});
    ASSERT_EQ(bootloader_.apply_update(v2, Bootloader::SLOT_A_ADDR),
              Bootloader::UpdateResult::Success);

    auto v1 = make_image(1, {0xBB});
    EXPECT_EQ(bootloader_.apply_update(v1, Bootloader::SLOT_A_ADDR),
              Bootloader::UpdateResult::VersionDowngrade);
}

TEST_F(BootloaderTest, WriteFailureDetected) {
    flash_.inject_write_failure();
    auto image = make_image(1, {0x01, 0x02});
    EXPECT_EQ(bootloader_.apply_update(image, Bootloader::SLOT_A_ADDR),
              Bootloader::UpdateResult::WriteError);
}

TEST_F(BootloaderTest, OversizedImageRejected) {
    std::vector<uint8_t> big_payload(Bootloader::MAX_FW_SIZE + 1, 0xAA);
    auto image = make_image(1, big_payload);
    EXPECT_EQ(bootloader_.apply_update(image, Bootloader::SLOT_A_ADDR),
              Bootloader::UpdateResult::TooLarge);
}

```

### Q3: Test A/B slot switching and rollback

**Answer:**

```cpp

TEST_F(BootloaderTest, ABSlotIndependent) {
    auto imageA = make_image(1, {0xAA});
    auto imageB = make_image(2, {0xBB});

    ASSERT_EQ(bootloader_.apply_update(imageA, Bootloader::SLOT_A_ADDR),
              Bootloader::UpdateResult::Success);
    ASSERT_EQ(bootloader_.apply_update(imageB, Bootloader::SLOT_B_ADDR),
              Bootloader::UpdateResult::Success);

    // Verify both slots have correct data
    FirmwareHeader hdrA, hdrB;
    std::array<uint8_t, sizeof(FirmwareHeader)> buf{};

    flash_.read(Bootloader::SLOT_A_ADDR, buf);
    std::memcpy(&hdrA, buf.data(), sizeof(hdrA));
    EXPECT_EQ(hdrA.version, 1);

    flash_.read(Bootloader::SLOT_B_ADDR, buf);
    std::memcpy(&hdrB, buf.data(), sizeof(hdrB));
    EXPECT_EQ(hdrB.version, 2);
}

TEST_F(BootloaderTest, EmptyImageRejected) {
    std::vector<uint8_t> empty;
    EXPECT_EQ(bootloader_.apply_update(empty, Bootloader::SLOT_A_ADDR),
              Bootloader::UpdateResult::BadMagic);
}

```

---

## Notes

- **Bootloader code must be tested more rigorously** than application code — a bug bricks the device
- Flash simulator should model flash physics: erase sets to 0xFF, write can only clear bits
- Test power-loss scenarios by injecting write failures at different points in the update sequence
- A/B slot (dual bank) updates enable rollback — test both slot A→B and B→A transitions
- CRC verification after writing catches bit-rot and flash programming errors
- Version downgrade prevention is a security requirement — always test it
- Integration tests should verify the full sequence: download → verify → erase → write → verify → boot
