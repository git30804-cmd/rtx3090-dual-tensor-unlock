# RTX 3090 × 2 Dual-GPU Tensor Unlock on Windows — Field Success Story
# RTX 3090 듀얼 GPU 텐서 언락 (Windows) — 실전 성공 사례

RTX 30xx FP16/BF16 **tensor speed-selector unlock** applied pre-driver via a USB UEFI app, ported from an MSI RTX 3080 project to a **dual reference-RTX 3090** rig — and verified with interleaved A/B GEMM benches on both cards.

MSI RTX 3080용으로 검증된 프로젝트를 **레퍼런스 RTX 3090 2장** 환경에 포팅하고, 두 카드 모두에서 인터리브드 A/B 벤치로 실증한 기록입니다.

---

## Results (실측 결과)

One cold-boot session, two `U` keypresses → both cards unlocked:

| Metric | Locked GPU1 (baseline) | Unlocked GPU0 | Unlocked GPU1 |
|---|---:|---:|---:|
| FP16 tensor / FP32 accum | 74.2–74.3 TFLOPS | **110.8 / 112.7 / 112.8** | **112.5 / 111.0 / 112.8** |
| BF16 tensor | ~74 | **~116.5** | **~117** |
| FP32 (control) | ~25.7 | ~25 (unchanged ✓) | ~25.6 (unchanged ✓) |

- Locked ceiling ≈ ½ of GA102's full tensor rate; FP16T > 71 TFLOPS proves the selector is released. Gain: **+37~50%**.
- Consistent with community field data on unlocked 3090 (≈116 TFLOPS @ ~252 W).
- Unlock is **volatile** (no vBIOS flash): a full power cut resets it; reapply per cold boot in under a minute.

락 상태 상한은 GA102 텐서 성능의 약 절반(셀렉터 반속). FP16T가 71을 넘는 순간 언락 실증. 이득 +37~50%, 커뮤니티 실측(≈116@252W)과 일치. **vBIOS 플래시가 아니라 휘발성** — 완전 전원차단이면 초기화, 부팅마다 USB로 1분 만에 재적용.

## Hardware / Environment (환경)

- ASUS BD790i (AMD 7945HX, ITX), **2× reference RTX 3090** on PCIe risers — GPU0 = PCI bus 1 (`0:1:0.0`), GPU1 = bus 3 (`0:3:0.0`)
- Windows 11, NVIDIA driver current branch; UEFI console is the iGPU's Radeon output (monitor on motherboard ports)
- Secure Boot OFF, FAT32 USB with `EFI/BOOT/BOOTX64.EFI`

## Method (방법)

The EFI app runs **before the NVIDIA driver binds**, releases firmware write protection (WPR), loads GSP payload assets, writes the tensor speed selectors directly (`SS0 0x5723…→0x88888888`, `SS1 0x7→0x8`), then performs a Function-Level Reset and verifies the values survive post-FLR. No reboot between EFI and Windows; never mid-way reboots.

EFI 앱이 **드라이버 바인딩 이전**에 PCI config/GSP 페이로드 경로로 텐서 셀렉터를 직접 쓰고 FLR 후 유지를 검증합니다(vBIOS 미플래시, WPR 해제 포함). U 적용 후 재부팅 없이 그대로 Windows로 부팅하는 것이 핵심 조건.

### What we changed for the 3090 pair (포팅 시 수정한 것)

1. **Board identity**: vendor/device `2204:10de`, subsystem pins, MSI-X/identity decoys — every hardcoded ID replaced and gated behind `GU_TARGET_BUS`-style macros in one header.
2. **Bus targeting for two identical cards**: FindTarget matches segment 0 / bus N / dev 0 via `GetLocation`; per-card EFI variants first (`-DGU_TARGET_BUS=1|3`).
3. **Console/display gate fix**: the vendor `DisplayCheck` demands an Intel-iGPU ACPI display root; on this AMD iGPU machine that aborts before the READY prompt (symptom: boot flashes past USB in seconds). Fix = wrap path classifier, accept any END_ENTIRE-terminated path containing ≥1 `HW(PCI)` node as fallback route 9.
4. **Multi-bus single-pass build** (`-DGU_MULTI_BUSES`): one boot unlocks both cards sequentially (READY → U → `==== next target ====` → READY again → U). Key enablers: full state re-init via GuInit between passes, runtime bus global instead of compile-time gate; report shows `FINAL status=0 … clean=1` per pass.

### Field rounds (현장 라운드) — all evidence on USB as `RUNxx/REPORT.TXT`

| Round | Result |
|---|---|
| RUN00–01 (v1 EFI) | target OK, **display gate abort** → fixed with route-9 fallback |
| RUN02 / 04 (bus1 v5) | full success: FWSEC/WPR, Falcon load, SS writes held post-FLR — but unlock lost during Windows boot rounds before this design existed |
| RUN05–06 | **FP16T ~111 vs locked sibling ~74 (+50%)** with a login-triggered auto-bench (one-shot Run-key batch that tree-kills the LLM server and benches in the clean first minute) — proved consumption AND which boot-time ops kill selectors |
| RUN08 (multi EFI `6e68…`) | **both cards, one cold boot**: pass1 `0:1:0.0` FINAL clean → next target → pass2 `0:3:0.0` FINAL clean; auto-bench confirms both ~112. First try. |

See [`report-RUN08.txt`](report-RUN08.txt) (raw EFI report, both FINAL lines) and [`bench-v5ab-tail.txt`](bench-v5ab-tail.txt) (timestamped A/B bench).

## Pitfalls learned (함정 정리)

- **Warm reboot fails the method**: the card must be untouched by the driver since power-on → every attempt = PSU-off cold boot. Unlock itself also resets on full power cut only, but applying requires the driver-not-yet-bound window.
- **Selector killers during Windows boot**: NVAPI performance writes (`-pl`, `-lgc`) and even unlogged autostart stack that loads tens of GB onto the GPU can wipe SS values silently → we bench inside ~60 s of login via a self-deleting Run-key batch, before any autostart touches the cards.
- **Console wiring matters**: if your monitor were on the dGPU being FLR'd, screen would go black mid-run; keep console on iGPU output (route-9 fallback makes the app tolerate AMD consoles regardless).
- **Power-limit policy noise**: locked-card numbers move with PL (~55 @230 W vs ~74 @350 W) — sibling A/B in the same session under identical policy is the only trustworthy comparison; FP16T > 71 (not raw absolute TFLOPS) is the unlock criterion.
- Unlock helps GPU-internal tensor GEMM workloads (ComfyUI video generation, LLM prefill). PCIe/RAM-offloaded expert streaming or decode barely moves — don't expect end-to-end ratios to match the bench delta.

## Original sources / credit (최초 출처)

This port stands on two upstream GitHub projects — they are the originators; we only contributed board-port patches and dual-card field verification:

- **[Zagitta/gpu-unlock](https://github.com/Zagitta/gpu-unlock)** at commit `0ebdb1c6308b6a2072be5bf04513472942541956` — reusable core (GPL-2.0-only; its tree carries CMP40HX MIT and NVIDIA driver asset licenses).
- **[rough2515-hub/rtx3080-unlock-public](https://github.com/rough2515-hub/rtx3080-unlock-public)** — the MSI RTX 3080 Windows UEFI port (v0.10) this case study is built on.

## Reproduce (재현)

Contents of a ready FAT32 USB root: `EFI/BOOT/BOOTX64.EFI` (multi-bus build for bus 1+3), `RTX3080-UNLOCK/ASSETS/*` (5 assets per `assets.json` manifest). Cold boot → UEFI USB → READY = press `U`, second target announces itself, press `U` again → Windows Boot Manager. Verify with a big-square FP16/BF16 cuBLAS matmul on the *same power policy* as your locked baseline (see pitfalls).

---
*Written automatically from field logs (2026-10-19) by the operator's AI agent, under account `git30804-cmd`.*
