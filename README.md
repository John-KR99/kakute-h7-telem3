# PX4 Kakute H7 — UART3 as TELEM3

Custom PX4 firmware build workflow that exposes **UART3 (R3/T3, `/dev/ttyS2`) as `TELEM3`** on the **Holybro Kakute H7 V1.5** flight controller.

The goal is to make the otherwise console-reserved UART3 available for a configurable serial service, such as MAVLink telemetry or a supported GPS driver, without modifying the hardware or bootloader.

> **Status:** Community-maintained, board-specific PX4 customization. The GitHub Actions build and generated configuration checks have passed, and the author has performed limited testing on a Kakute H7 V1.5. This is **not** an official PX4/Holybro release or a general flight-safety certification.

## Why this exists

In the pinned upstream Kakute H7 target, USART3 (`/dev/ttyS2`) is reserved for the NuttX serial console. Merely selecting another MAVLink port in QGroundControl does not free that device.

This project changes the **build-time board configuration**, so PX4 can present the port as `TELEM3` and allow a supported service to use it after the firmware is flashed.

## Compatibility

| Item | Target |
| --- | --- |
| Flight controller | **Holybro Kakute H7 V1.5** |
| PX4 build target | `holybro_kakuteh7_default` |
| PX4 source | Pinned upstream commit `d6f12ad1c4f70ad3230afd7d86e971421e02fef4` |
| Physical UART | UART3, labeled **R3 / T3** |
| NuttX device | `/dev/ttyS2` |
| New PX4 port name | `TELEM3` |
| Other boards | **Not validated** for Kakute H7 Mini, Kakute H7 V2, or other variants |

Check your board revision and pinout before building or flashing. Do **not** assume this image is interchangeable with `holybro_kakuteh7v2_default` or `holybro_kakuteh7mini_default`.

## What the workflow changes

The [build workflow](.github/workflows/build_kakute_h7_telem3.yml) checks out the pinned PX4 source, applies a narrowly scoped change, builds the firmware, validates the result, and uploads the outputs.

Key changes include:

1. Add `CONFIG_BOARD_SERIAL_TEL3="/dev/ttyS2"` to `boards/holybro/kakuteh7/default.px4board`.
2. Disable `CONFIG_USART3_SERIAL_CONSOLE` in the board's NuttX configuration and enable `CONFIG_NO_SERIAL_CONSOLE=y`.
3. Keep console diagnostics through the configured syslog/RAMLOG backend instead of assigning a physical UART console.
4. Keep STM32H7 serial-device ordering stable with `CONFIG_STM32H7_SERIAL_DISABLE_REORDERING=y`, preserving the expected `/dev/ttyS*` assignments.

**Important:** UART3 is no longer available as the original serial debug console in this build. The change does **not** automatically launch MAVLink, GPS, or any other protocol on TELEM3.

## Build and download a `.px4` firmware file

This repository contains the **build recipe**, not a complete PX4 source fork and not a permanently hosted firmware binary.

1. Open [GitHub Actions](https://github.com/John-KR99/kakute-h7-telem3/actions) for this repository.
2. Select **Kakute H7 UART3 Telem3 R4**.
3. Choose **Run workflow** on the `main` branch.
4. Wait until the workflow finishes successfully.
5. Under the completed run's **Artifacts**, download `kakute_h7_uart3_telem3_unflighttested`.
6. Extract `kakute_h7_uart3_telem3_d6f12ad1c4f7.px4`; check `SHA256SUMS.txt` and the supplied patch/validation files if needed.

GitHub Actions artifacts have a **30-day retention period** in this workflow. If an older run no longer offers a download, start a new run. The artifact name retains the original conservative `unflighttested` label; it is **not** a claim that the later author-reported hardware tests never happened.

The workflow uses the pinned PX4 source and PX4 development container to run `make holybro_kakuteh7_default`. Its automated checks inspect the generated NuttX configuration, `SER_TEL3_BAUD`, MAVLink port-selection metadata (`TELEM3 = 103`), serial startup scripts, and `.px4` package integrity.

The [September 7, 2026 build](https://github.com/John-KR99/kakute-h7-telem3/actions/runs/34091755804) completed successfully. A green build confirms compilation and those static checks; it does **not** by itself prove electrical compatibility or flight safety.

## Flash and configure

1. **Back up** existing PX4 parameters and verify your exact flight-controller revision.
2. Use QGroundControl's custom firmware option to flash the downloaded `.px4` file to a compatible Kakute H7 V1.5. This workflow does **not** replace the bootloader.
3. **Remove propellers** for initial configuration and testing. Keep a working USB/QGroundControl connection available.
4. Assign **one** supported service to TELEM3 and configure the connected device's serial baud rate. Reboot after parameter changes.
5. Verify boot, sensors, RC input, existing GPS/telemetry links, and UART3 TX/RX before any flight testing.

### Example: use TELEM3 for MAVLink

- In QGroundControl, choose an **unused** `MAV_n_CONFIG` instance and select **TELEM3** (the generated parameter metadata identifies it with value `103`).
- Set an appropriate `MAV_n_MODE` and match `SER_TEL3_BAUD` to the connected device.
- Reboot and verify actual MAVLink communication over the physical R3/T3 pins.
- Do not reassign a MAVLink instance already required by Herelink or another telemetry connection.

For a **GPS or another serial device**, do not enable MAVLink on the same UART. Select the corresponding supported PX4 driver/port configuration instead, if that driver is included in the build. The correct setup depends on the device and protocol.

If communication fails, first verify the board revision, UART3 pinout, wiring (including TX/RX direction and ground), electrical levels, baud rate, service assignment, and reboot state.

## Verification and limitations

- **Automated:** The pinned workflow has a successful GitHub Actions run and checks the generated configuration and firmware package.
- **Hardware:** After the initial build, the author reported successful boot and UART3-connected GPS/RTK operation on a Kakute H7 V1.5, plus limited indoor/outdoor flight tests. These are project-specific observations, **not** independent testing or a guarantee for other airframes.
- **Not established here:** Long-duration UART reliability, every supported peripheral/protocol, other Kakute H7 revisions, and behavior under all environmental conditions.
- **Risk:** Custom flight-controller firmware can affect control links, telemetry, and aircraft safety. Back up your configuration, test without propellers first, and proceed at your own risk.

For reproducibility, inspect the [workflow source](.github/workflows/build_kakute_h7_telem3.yml) and the diagnostic patch/configuration produced by a fresh build. No upstream PX4 pull request or official support is implied.

## 한국어 요약

이 저장소는 **Holybro Kakute H7 V1.5**의 기본 PX4 펌웨어에서 NuttX 콘솔로 사용되던 **UART3(R3/T3, `/dev/ttyS2`)**를 **TELEM3**으로 활성화하는 커스텀 빌드 워크플로입니다. 단순 파라미터 변경이 아니라 보드/NuttX 설정을 수정해 빌드합니다.

**Actions → Kakute H7 UART3 Telem3 R4 → Run workflow**를 실행하면 `.px4` 파일과 검증 자료가 생성됩니다. 빌드 산출물은 30일 후 만료되므로 필요하면 다시 실행하세요. 실기체에서 UART3 GPS/RTK 및 제한적인 비행 시험이 이루어졌지만, 다른 보드나 기체에서의 안전성을 보장하지는 않습니다. 먼저 프로펠러를 제거하고 USB 연결을 유지한 상태에서 점검하세요.
