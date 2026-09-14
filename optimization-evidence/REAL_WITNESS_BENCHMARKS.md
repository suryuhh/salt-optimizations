# Real-witness and direct-base benchmark evidence

This evidence supports the four claims in the [maintainer report](../OPTIMIZATION_REPORT.md). It records per-PR comparisons rather than assigning historical whole-stack effects to a consolidated PR.

## Populations and integrity

| Population | Fixed block positions | Transactions/block | Commitments/witness | Purpose |
|---|---|---:|---:|---|
| Dense | 6,900,001–6,900,127 | 10092 average | 49,635–50,342 | High-load scaling |
| Ordinary head | 26,000,001–26,000,127 | 26.25 average | 377–1,973 | Current small-block behavior |

Eight evenly spaced positions were fixed before running. The harness decoded the `SaltWitness` prefix from the validator's legacy-bincode payload, verified each proof, checked semantic codec round trips and emitted proof identities. Every computed root matches the preceding block header. All source arms emitted the same block/root/key/commitment identities.

## Direct results

The report's tables are generated from the complete round tables below. Values are medians of three alternating rounds. Each process first decodes and verifies all inputs outside timing, then performs fixed repeated work. `CLOCK_PROCESS_CPUTIME_ID` records process CPU; `/proc/self/status` records RSS and high-water RSS.

- PR #2 versus upstream main plus the pinned #152 verifier subset: dense decode 169.53 → 22.80 ms.
- PR #3 versus PR #2: dense one-proof verification 125.97 → 84.10 ms; concurrent verification rate 17.60 → 38.44 verified proofs/s.
- PR #4 versus PR #3: the complete five-thread 10,000-KV table appears in the report and below.
- PR #5 versus PR #4: dense concurrent verification 38.23 → 62.87 verified proofs/s and peak 726.01 → 273.09 MiB.
- Upstream main plus the pinned #152 verifier subset versus PRs #2–#5 combined: dense decode-plus-check 215.69 → 29.78 ms; ordinary head 5.479 → 2.521 ms.

A clean-clone rerun from `megaeth-labs/salt` using the public script reproduced the combined dense and ordinary-head results at 7.37× and 2.17×. Every reported direction reproduced; the largest change in a displayed component percentage was 3.7 points (PR #5 dense one-proof latency, -40.2% to -36.5%).

## Scope checks

The repaired MSM result is equal to arkworks at 0, 127, 128, 1,023, 1,024, 4,351, 8,191, 8,192, 8,193 and 16,385 points in one-, two- and eight-thread pools. The generated #152 proof cases use 52, 760 and 4,351 stored commitments; the two smaller cases and the 4,351 case remain below the multi-thread vector cutoff. The complete round values are below.

The direct eight-thread crossover check leaves 1,024–8,191 points on the scalar path. Median scalar/vector times are 7.042/5.519 ms at 8,192 points, 13.080/9.191 ms at 16,384, and 27.729/11.698 ms at 50,000.

The trie matrix uses SALT's existing `update 10000 KVs` benchmark and its 1/2/4/8/16 thread counts. Only the per-case Criterion measurement window changes from 30 to three seconds; the setup and timed implementation are unchanged. Three alternating rounds were run.

The existing repeated-polynomial 16,000-query case measures 9.063 ms on PR #5's base and 9.294 ms on PR #5. It clones one polynomial, so commitment coalescing leaves a small MSM. This is useful regression coverage but not representative of the mainnet proofs above.

## Reproduce

The two source blocks below contain the complete reproduction script and benchmark harness. The script fetches the five pinned source revisions, verifies the corpus and harness hashes, builds immutable binaries, then runs the same three alternating rounds and prints the report’s real-witness tables. It requires Linux x86-64 with AVX-512F and AVX-512IFMA and rejects other machines before building.

```sh
curl -fL https://raw.githubusercontent.com/suryuhh/salt-optimizations/codex/docs/optimization-report/optimization-evidence/REAL_WITNESS_BENCHMARKS.md -o salt-benchmark.md
python3 - <<'PY'
from pathlib import Path
import hashlib, re
page = Path("salt-benchmark.md").read_text()
for name, language, expected in [
    ("reproduce-real-witness.sh", "bash", "11611948110bfa7a896984b6e51bf7ec4105c05ad9b600f77971a7de701e875c"),
    ("real-witness-harness.rs", "rust", "ab8fda7b37c161ce36c2744ee7da95ce6b708eabc2e66bcd133bda0b8b8534dd"),
]:
    marker = "<!-- file:" + name + " -->\n```" + language + "\n"
    match = re.search(re.escape(marker) + r"(.*?)\n```", page, re.S)
    if match is None:
        raise SystemExit("missing benchmark source: " + name)
    content = (match.group(1) + "\n").encode()
    if hashlib.sha256(content).hexdigest() != expected:
        raise SystemExit("checksum mismatch: " + name)
    Path(name).write_bytes(content)
PY
bash reproduce-real-witness.sh /path/to/salt /tmp/salt-witness-reproduction
```

The script writes `runs.jsonl`, `summary.json`, and `SUMMARY.md` into the local output directory you choose. These are benchmark measurements produced on your machine. The [16-witness corpus](https://github.com/suryuhh/salt-optimizations/releases/download/optimization-evidence-2026-09-13/salt-witness-benchmark-corpus-v1.tar.zst) is 23,605,740 bytes, SHA-256 `ac713399727c5e5d50f4fb9f8fe9a7877fc435aa4e46ea4bf7cc8e66ee71f8ca`. It contains only witness payloads and their block/hash manifests. Expected roots and block identities appear below.

## Exact revisions

| Arm | Commit |
|---|---|
| Upstream main + pinned #152 verifier subset | `713511618e1db1cc5123398d35a8996352fa6a89` |
| PR #2 | `c836c75f2dd0f155ca2b56875669c35a16943c45` |
| PR #3 | `35436e1a689010bc9cc8ac566648916c0ef5b7ca` |
| PR #4 | `e7e1e43958802e67c0d1417ab114f964a3a9b0b7` |
| PR #5 | `52a460c12b03e6c352f7c9a4cb207e38fc244bbf` |

The reference assembles the verifier-relevant subset of [#152 at `6cfc1cd`](https://github.com/megaeth-labs/salt/commit/6cfc1cd9fad0b3ea2551899b6eeabe25b0b355a1): shared CRS, fixed-base proof work, scalar windowed MSM and the `Z = 1` normalization skip. It excludes #152's node-polynomial cache and refresh path, which affect witness creation rather than the supplied-witness verification timed here.

The pinned snapshots above contain the same source, tests, and build inputs as the published PR heads. Compared with the original timed revisions, only one documentation comment differs; the numeric observations below were preserved unchanged.

## Every recorded round

Elapsed milliseconds per completed item equal 1,000 divided by completion rate. With concurrency eight, that is an inverse-throughput measure, not individual-request latency. Process CPU and peak RSS are recorded separately.

<details>
<summary>PR #2 · dense decoding</summary>

| Arm | Round | Elapsed ms/item | CPU ms/item | Items/s | Peak MiB |
|---|---:|---:|---:|---:|---:|
| control | 1 | 169.275634625 | 1288.72510553125 | 5.907524743388626 | 246.91015625 |
| control | 2 | 169.5306484375 | 1292.7631318125 | 5.898638442173275 | 246.68359375 |
| control | 3 | 170.12604825 | 1292.97851015625 | 5.877994641540732 | 246.76953125 |
| treatment | 1 | 23.424694406249998 | 114.15473174999998 | 42.68999128258372 | 250.23828125 |
| treatment | 2 | 22.8029419375 | 113.13361090624998 | 43.8539905395047 | 239.12109375 |
| treatment | 3 | 22.41104128125 | 112.54770909375003 | 44.62086287961288 | 239.21484375 |

</details>

<details>
<summary>PR #3 · dense, one proof</summary>

| Arm | Round | Elapsed ms/item | CPU ms/item | Items/s | Peak MiB |
|---|---:|---:|---:|---:|---:|
| control | 1 | 127.78024465624999 | 451.14724718750006 | 7.825935868961322 | 250.17578125 |
| control | 2 | 125.97189381250001 | 446.5551702499999 | 7.938278688486077 | 250.0546875 |
| control | 3 | 123.13256521875 | 442.1489051875001 | 8.121328409129294 | 250.50390625 |
| treatment | 1 | 79.36428709375 | 200.49592071875003 | 12.600125782251887 | 265.09375 |
| treatment | 2 | 87.20106434374999 | 212.12067424999998 | 11.467749935459056 | 262.328125 |
| treatment | 3 | 84.09961553125001 | 205.59450493749998 | 11.89066077987499 | 257.79296875 |

</details>

<details>
<summary>PR #3 · dense, eight concurrent proofs</summary>

| Arm | Round | Elapsed ms/item | CPU ms/item | Items/s | Peak MiB |
|---|---:|---:|---:|---:|---:|
| control | 1 | 56.079018812499996 | 438.60423468749997 | 17.831981036321203 | 713.47265625 |
| control | 2 | 56.81974996875 | 443.91723009375005 | 17.59951426308607 | 710.859375 |
| control | 3 | 56.884897343750005 | 442.80783771874997 | 17.579358435985135 | 719.2265625 |
| treatment | 1 | 26.487996906249997 | 203.22922834375 | 37.75294913916439 | 719.80859375 |
| treatment | 2 | 25.66593075 | 198.75550796875 | 38.9621560870143 | 729.84765625 |
| treatment | 3 | 26.01584753125 | 198.98434065625003 | 38.43810964831375 | 702.640625 |

</details>

<details>
<summary>PR #3 · ordinary head, one proof</summary>

| Arm | Round | Elapsed ms/item | CPU ms/item | Items/s | Peak MiB |
|---|---:|---:|---:|---:|---:|
| control | 1 | 4.361882925000001 | 22.85239162625 | 229.25878965447015 | 11.0078125 |
| control | 2 | 4.45645114375 | 23.06257514125 | 224.3937985054456 | 10.48828125 |
| control | 3 | 4.3798769887499995 | 22.804083065 | 228.3169145089155 | 10.79296875 |
| treatment | 1 | 4.3456968475 | 22.7127613175 | 230.11269195532626 | 11.01171875 |
| treatment | 2 | 4.49641425625 | 23.012396016250005 | 222.39943719820823 | 11.1328125 |
| treatment | 3 | 4.43124865625 | 22.84855409625 | 225.67002611996563 | 10.80078125 |

</details>

<details>
<summary>PR #5 · dense, one proof</summary>

| Arm | Round | Elapsed ms/item | CPU ms/item | Items/s | Peak MiB |
|---|---:|---:|---:|---:|---:|
| control | 1 | 85.9922539375 | 208.3249146875 | 11.62895440241408 | 260.83984375 |
| control | 2 | 87.85592625 | 213.09041440624998 | 11.382271437835989 | 263.43359375 |
| control | 3 | 87.1766935625 | 209.8025624375 | 11.470955815536469 | 263.50390625 |
| treatment | 1 | 51.670916625000004 | 143.15899953125 | 19.35324676466391 | 170.078125 |
| treatment | 2 | 52.16419553125 | 143.76686506250002 | 19.170237167770182 | 170.08984375 |
| treatment | 3 | 52.360956906249996 | 144.25105603125 | 19.098199480778327 | 168.72265625 |

</details>

<details>
<summary>PR #5 · dense, eight concurrent proofs</summary>

| Arm | Round | Elapsed ms/item | CPU ms/item | Items/s | Peak MiB |
|---|---:|---:|---:|---:|---:|
| control | 1 | 26.34337096875 | 203.64813128125 | 37.960214020683104 | 710.37109375 |
| control | 2 | 26.154965375 | 200.62629037500002 | 38.233657956046905 | 726.01171875 |
| control | 3 | 25.9864943125 | 200.913235875 | 38.481527672587255 | 728.24609375 |
| treatment | 1 | 15.8797718125 | 126.91873359375 | 62.97319708415678 | 273.09375 |
| treatment | 2 | 15.905353968750001 | 127.14175146875 | 62.871911053645654 | 274.24609375 |
| treatment | 3 | 15.99359696875 | 127.14096950000001 | 62.52502185430251 | 271.4921875 |

</details>

<details>
<summary>PR #5 · ordinary head, one proof</summary>

| Arm | Round | Elapsed ms/item | CPU ms/item | Items/s | Peak MiB |
|---|---:|---:|---:|---:|---:|
| control | 1 | 4.55029184375 | 23.33925087125 | 219.7661236550219 | 11.11328125 |
| control | 2 | 4.4561461425 | 23.055684855000003 | 224.40915715546464 | 11.0078125 |
| control | 3 | 4.46858363375 | 23.189798179999997 | 223.78455500916917 | 11.13671875 |
| treatment | 1 | 3.7876129874999998 | 20.870172337499998 | 264.01852652323026 | 6.10546875 |
| treatment | 2 | 3.76351379625 | 20.669844980000004 | 265.70913623231814 | 6.09375 |
| treatment | 3 | 3.823210755 | 20.90220747875 | 261.56026023210956 | 6.05078125 |

</details>

<details>
<summary>PR #5 · ordinary head, eight concurrent proofs</summary>

| Arm | Round | Elapsed ms/item | CPU ms/item | Items/s | Peak MiB |
|---|---:|---:|---:|---:|---:|
| control | 1 | 2.3760019375 | 18.2666793 | 420.87507767446846 | 23.0078125 |
| control | 2 | 2.3321585275000003 | 17.982459426250003 | 428.7873179324427 | 21.7421875 |
| control | 3 | 2.347871885 | 18.0529766075 | 425.9176177323662 | 23.375 |
| treatment | 1 | 2.2211387662499997 | 17.32605709125 | 450.2195068560814 | 13.140625 |
| treatment | 2 | 2.20063767 | 17.215561085 | 454.4137427221265 | 13.08203125 |
| treatment | 3 | 2.1829492975 | 17.058300045 | 458.0958436117777 | 13.17578125 |

</details>

<details>
<summary>All four PRs · dense decode plus verification</summary>

| Arm | Round | Elapsed ms/item | CPU ms/item | Items/s | Peak MiB |
|---|---:|---:|---:|---:|---:|
| control | 1 | 215.03021625 | 1709.7027900937499 | 4.650509204889478 | 789.23828125 |
| control | 2 | 215.7585456875 | 1710.1475504375 | 4.634810625060378 | 793.69140625 |
| control | 3 | 215.69371843750002 | 1715.7619 | 4.636203628200525 | 785.30859375 |
| treatment | 1 | 29.686490406249998 | 235.07933934375 | 33.685356076630285 | 341.734375 |
| treatment | 2 | 29.7986085625 | 236.69823218750003 | 33.5586139165722 | 346.53515625 |
| treatment | 3 | 29.7821025625 | 236.70642949999998 | 33.57721295537896 | 346.21484375 |

</details>

<details>
<summary>All four PRs · ordinary-head decode plus verification</summary>

| Arm | Round | Elapsed ms/item | CPU ms/item | Items/s | Peak MiB |
|---|---:|---:|---:|---:|---:|
| control | 1 | 5.5030731062500005 | 41.36021043875 | 181.71664826045486 | 22.33984375 |
| control | 2 | 5.4792263475 | 41.22452002 | 182.5075177732471 | 22.4296875 |
| control | 3 | 5.473658017500001 | 41.12766801875 | 182.69318192749148 | 23.17578125 |
| treatment | 1 | 2.52000490875 | 19.6231454525 | 396.8246238440983 | 12.5625 |
| treatment | 2 | 2.554049905 | 19.6911973825 | 391.53502758200807 | 12.37890625 |
| treatment | 3 | 2.52105555875 | 19.550235623749998 | 396.6592471670176 | 12.421875 |

</details>

<details>
<summary>Generated-proof verification</summary>

| Source | Queries | Round 1 ms | Round 2 ms | Round 3 ms |
|---|---:|---:|---:|---:|
| decoder-final | 16 | 3.208228990797828 | 3.341575982178932 | 3.4101119812592118 |
| decoder-final | 256 | 3.8948931237276785 | 3.9733394806858175 | 3.9006572437078373 |
| decoder-final | 2048 | 13.343435270303289 | 13.789774125133222 | 13.097825215011337 |
| msm-final | 16 | 3.289843848628427 | 3.3956830150329442 | 3.3082069810345804 |
| msm-final | 256 | 3.934303420955832 | 3.9240479228105594 | 3.948192100634076 |
| msm-final | 2048 | 13.5688219603288 | 13.631063561284014 | 13.702963661251527 |

</details>

<details>
<summary>SALT 10,000-KV trie update</summary>

| Source | Threads | Round 1 ms | Round 2 ms | Round 3 ms |
|---|---:|---:|---:|---:|
| msm-final | 1 | 193.7177564 | 191.67973606 | 195.80549491 |
| msm-final | 2 | 108.03583219 | 108.92108976 | 107.80074591 |
| msm-final | 4 | 64.77349505 | 65.10452028 | 64.03281065 |
| msm-final | 8 | 42.364983009999996 | 42.10029351 | 42.415290979999995 |
| msm-final | 16 | 32.94690148 | 32.07038893 | 32.6733982 |
| committer-final | 1 | 120.40703959 | 121.06705609999999 | 121.83979233 |
| committer-final | 2 | 69.77178568000001 | 69.29102515999999 | 69.55803148000001 |
| committer-final | 4 | 43.95022152000001 | 44.554885049999996 | 44.66248546 |
| committer-final | 8 | 31.42330154 | 30.9558816 | 31.4397493 |
| committer-final | 16 | 30.22134378 | 30.547978609999998 | 30.92876097 |

</details>

<details>
<summary>Repeated-polynomial regression check</summary>

| Source | Round 1 ms | Round 2 ms | Round 3 ms |
|---|---:|---:|---:|
| committer-final | 9.215661878452382 | 9.063045303796297 | 9.018988474947092 |
| storage-16k-final | 9.257808314351852 | 9.294005342566136 | 9.345430467519842 |

</details>

<details>
<summary>MSM crossover</summary>

| Points | Decoder base ms | MSM PR ms |
|---:|---:|---:|
| 1024 | 1.1227757 | 1.09956255 |
| 2048 | 2.0577385 | 2.0476399 |
| 4096 | 4.52278215 | 4.5144321 |
| 8191 | 5.8105712 | 5.810674100000001 |
| 8192 | 7.042487799999999 | 5.5187749 |
| 16384 | 13.079620499999999 | 9.1910494 |
| 50000 | 27.729203000000002 | 11.69793075 |

</details>

## Input identities

<details>
<summary>dense blocks and expected SALT roots</summary>

| Block | Transactions | Parent block hash | Expected SALT state root |
|---:|---:|---|---|
| 6900001 | 10072 | 0x0ee303d419f4c268278f59eaa31f8943893c3f7976f3d351a3abeab94d842ecc | 0x5704d6556f422986a1f3c8c15a1c80de405a2206af84d0f277bb1d006d186911 |
| 6900019 | 10130 | 0xe0c0ba5588578f422c5f2359840b3f3e76dd11ce53e06b67bfc2400e00717320 | 0xcae56db39e66a6d64e426d71d53130856108ebdbc77bed426e15fecadec4dc03 |
| 6900037 | 10096 | 0xcb06abd722ff17fc86a430b80384d408631a086b0475a9779b218db5ba89e7da | 0x0202d006cf8dd554b8f4f3a98b595f8dd0237346ec9b29798ad8d23392a9a115 |
| 6900055 | 10053 | 0x3cdc194761bf7f16d8f9587351c1abf878ec5a56cf19838339be5df6aaba3fec | 0xb5580fbfac28015478db4004f0ab92b3080900a9e17d075048883779eedd7816 |
| 6900073 | 10064 | 0x4f9aeab74585b627c4241361e94be47013bd8d15f12ac78f88ccfa36e0733cc8 | 0xb64eb3e7e53a899086a8527317fc9a591fc1cba771639f90dcb79be7c2425209 |
| 6900091 | 10138 | 0x6444dfeb5d5ee79aa3a21601bc16f80480c47c1ed12af740b6bab58ed67f7217 | 0xffdf15697fef90df97b8865adfe4ed7feb0e58c15c15cb854206e29843992b02 |
| 6900109 | 10138 | 0x9cb098489ec24161659bfab68352dea354d1d0701d6173b4843798f3e3731bca | 0x1f95ab7bec85a83c09f87d8796a0a623e048cdab38990db7698dab45979e0f03 |
| 6900127 | 10048 | 0x8e9e3989e048f59adda096b5810ca06978b9e0bf91769d67cf4ac31b2261dcc1 | 0xe67e33fb188998c0a2d6b874e053df214c35c744d849842ebf7000927814281b |

</details>

<details>
<summary>head blocks and expected SALT roots</summary>

| Block | Transactions | Parent block hash | Expected SALT state root |
|---:|---:|---|---|
| 26000001 | 29 | 0x61fa103a3340cf4849cc8568826cfe3af8ea34eeb6eaefbdae19868631fa6617 | 0xae221437bffd41ebb61e62f00918ba2e2280256fb71b2cd3d175a6af483dd402 |
| 26000019 | 25 | 0x4bdad9e59c926e661232554572f8308968b7210a42a18fb410a8d4666995f963 | 0x6d81726c60fda52e024a450062776c121e8a940011bdb9df532c76ebc0f8d61c |
| 26000037 | 26 | 0x6d8f2e645d9e50c228ee7fdbd2c33629f5d22e7fe1142f920250f63e6a531a1a | 0x6980f644a72925ef10e994e64040551882fad3fa911616e3c27686db55469510 |
| 26000055 | 26 | 0xb43abe615b079b92f17ec4727fa0f529ad729d65bf3cc0afa12df1ecaf0da735 | 0x986782c45613d7abf57b46b342b21fb6e60324f3e4b9e0bcc5af73e33f6b8d02 |
| 26000073 | 26 | 0xcd6d4e87c3ef9094ff65a6a3e0f32b3b4d39fcabd938ad48970e856cb5267158 | 0x77f99d70d3b956b8e8c566d3b7a0df5d84d454cc46a16822f048d5a8b32a2515 |
| 26000091 | 28 | 0x10b3aeb3bc6eee0c93140f556123f5ae862ace9b03782806757cd06b3787a1dc | 0x026d94e681dae8f13ba4a1af656a25bee6189f877d3653700aaf6ff04952ae12 |
| 26000109 | 24 | 0xb04353a61da3301bbe51061ba2b53ac08c52be7fa0d9beb3f2b76c1e42643edd | 0x70f07adcc6c85d94edc7015d06f456fe17e7ad0bb83f921798e9ac6c69ff6500 |
| 26000127 | 26 | 0xbe015c8d1831c9af870c0f225b34872c99cd15feb9019d66e9bdefdb61edffc3 | 0x7fa66f73310bee3d430761e4115facb5db334ccb600d735697dc1a3add13771b |

</details>

<details>
<summary>reproduce-real-witness.sh</summary>

<!-- file:reproduce-real-witness.sh -->
```bash
#!/usr/bin/env bash
set -euo pipefail

if [[ $# -lt 1 || $# -gt 2 ]]; then
  echo "usage: $0 /path/to/salt [output-directory]" >&2
  exit 2
fi

for command in cargo curl git python3 tar; do
  command -v "$command" >/dev/null || {
    echo "required command is missing: $command" >&2
    exit 2
  }
done

if [[ $(uname -s) != Linux || $(uname -m) != x86_64 ]]; then
  echo "this benchmark requires Linux x86-64 because it reads /proc process memory" >&2
  exit 2
fi
if ! grep -qw avx512f /proc/cpuinfo || ! grep -qw avx512ifma /proc/cpuinfo; then
  echo "this benchmark requires AVX-512F and AVX-512IFMA to reproduce the reported vector paths" >&2
  exit 2
fi

repo=$(git -C "$1" rev-parse --show-toplevel)
out=${2:-${TMPDIR:-/tmp}/salt-witness-reproduction}
if [[ -e "$out" ]]; then
  echo "output directory already exists: $out" >&2
  exit 2
fi

fork_url=https://github.com/suryuhh/salt-optimizations.git
asset_url=https://github.com/suryuhh/salt-optimizations/releases/download/optimization-evidence-2026-09-13/salt-witness-benchmark-corpus-v1.tar.zst
harness_file="$(cd -- "$(dirname -- "${BASH_SOURCE[0]}")" && pwd)/real-witness-harness.rs"
asset_sha=ac713399727c5e5d50f4fb9f8fe9a7877fc435aa4e46ea4bf7cc8e66ee71f8ca
harness_sha=ab8fda7b37c161ce36c2744ee7da95ce6b708eabc2e66bcd133bda0b8b8534dd

labels=(reference decoder msm committer storage)
heads=(
  713511618e1db1cc5123398d35a8996352fa6a89
  c836c75f2dd0f155ca2b56875669c35a16943c45
  35436e1a689010bc9cc8ac566648916c0ef5b7ca
  e7e1e43958802e67c0d1417ab114f964a3a9b0b7
  52a460c12b03e6c352f7c9a4cb207e38fc244bbf
)

file_sha256() {
  if command -v sha256sum >/dev/null 2>&1; then
    sha256sum "$1" | cut -d' ' -f1
  else
    shasum -a 256 "$1" | cut -d' ' -f1
  fi
}

cleanup_worktrees() {
  for label in "${labels[@]}"; do
    source="$out/source/$label"
    if [[ -d "$source" ]]; then
      git -C "$repo" worktree remove --force "$source" >/dev/null 2>&1 || true
    fi
  done
  git -C "$repo" worktree prune >/dev/null 2>&1 || true
}
trap cleanup_worktrees EXIT

mkdir -p "$out"/{bin,source}
curl -fL "$asset_url" -o "$out/corpus.tar.zst"
cp "$harness_file" "$out/review_witness.rs"
test "$(file_sha256 "$out/corpus.tar.zst")" = "$asset_sha"
test "$(file_sha256 "$out/review_witness.rs")" = "$harness_sha"
tar --zstd -xf "$out/corpus.tar.zst" -C "$out"

# Fetch from the public fork explicitly. This works when the supplied clone's
# origin is megaeth-labs/salt and contains none of the review branches.
git -C "$repo" fetch --no-tags "$fork_url" \
  codex/refactor/verifier-prerequisites \
  codex/perf/batch-witness-decoding \
  codex/perf/vector-msm \
  codex/perf/batched-trie-commitment \
  codex/refactor/compact-witness-storage

for i in 0 1 2 3 4; do
  label=${labels[$i]}
  head=${heads[$i]}
  source="$out/source/$label"
  git -C "$repo" worktree add --detach "$source" "$head"
  mkdir -p "$source/salt/examples"
  cp "$out/review_witness.rs" "$source/salt/examples/review_witness.rs"
  CARGO_TARGET_DIR="$out/target" CARGO_INCREMENTAL=0 \
    cargo clean --manifest-path "$source/Cargo.toml" --release \
      -p banderwagon -p ipa-multipoint -p salt -p salt-macros
  CARGO_TARGET_DIR="$out/target" CARGO_INCREMENTAL=0 \
    cargo build --manifest-path "$source/Cargo.toml" --release \
      -p salt --example review_witness --locked -j8
  cp "$out/target/release/examples/review_witness" "$out/bin/$label"
done

run_case() {
  local population=$1 round=$2 corpus=$3 repeats=$4 mode=$5 concurrency=$6
  shift 6
  local case_labels=("$@")
  if [[ "$round" = 2 ]]; then
    local reversed=()
    for ((i=${#case_labels[@]}-1; i>=0; i--)); do
      reversed+=("${case_labels[$i]}")
    done
    case_labels=("${reversed[@]}")
  fi
  for label in "${case_labels[@]}"; do
    result=$(RAYON_NUM_THREADS=8 "$out/bin/$label" \
      "$corpus" "$mode" "$concurrency" "$repeats")
    printf '{"population":"%s","round":%s,"source":"%s","result":%s}\n' \
      "$population" "$round" "$label" "$result" >> "$out/runs.jsonl"
  done
}

: > "$out/runs.jsonl"
for round in 1 2 3; do
  # Run only source/mode combinations used by the report's real-witness tables.
  run_case dense "$round" "$out/corpus" 4 decode 1 reference decoder
  run_case dense "$round" "$out/corpus" 4 verify 1 decoder msm committer storage
  run_case dense "$round" "$out/corpus" 4 verify 8 decoder msm committer storage
  run_case dense "$round" "$out/corpus" 4 full 8 reference storage

  run_case head "$round" "$out/head-corpus" 100 verify 1 decoder msm committer storage
  run_case head "$round" "$out/head-corpus" 100 verify 8 committer storage
  run_case head "$round" "$out/head-corpus" 100 full 8 reference storage
done

python3 - "$out/runs.jsonl" "$out" <<'PY'
import json
import statistics
import sys
from collections import defaultdict
from pathlib import Path

runs_path, out_path = map(Path, sys.argv[1:])
runs = [json.loads(line) for line in runs_path.read_text().splitlines()]
grouped = defaultdict(list)
for row in runs:
    result = row["result"]
    key = (row["population"], result["mode"], result["concurrency"], row["source"])
    grouped[key].append(result)

for key, values in grouped.items():
    if len(values) != 3:
        raise SystemExit(f"expected three rounds for {key}, found {len(values)}")

def median(population, mode, concurrency, source):
    values = grouped[(population, mode, concurrency, source)]
    return {
        "wall_ms": statistics.median(v["wall_ms_per_check"] for v in values),
        "cpu_ms": statistics.median(v["cpu_ms_per_check"] for v in values),
        "proofs_per_second": statistics.median(v["checks_per_second"] for v in values),
        "peak_mib": statistics.median(v["final_memory_kib"]["VmHWM"] for v in values) / 1024,
    }

summary = {
    "stack_dense": {"base": median("dense", "full", 8, "reference"), "changed": median("dense", "full", 8, "storage")},
    "stack_head": {"base": median("head", "full", 8, "reference"), "changed": median("head", "full", 8, "storage")},
    "decode_dense": {"base": median("dense", "decode", 1, "reference"), "changed": median("dense", "decode", 1, "decoder")},
    "msm_dense_single": {"base": median("dense", "verify", 1, "decoder"), "changed": median("dense", "verify", 1, "msm")},
    "msm_dense_concurrent": {"base": median("dense", "verify", 8, "decoder"), "changed": median("dense", "verify", 8, "msm")},
    "msm_head_single": {"base": median("head", "verify", 1, "decoder"), "changed": median("head", "verify", 1, "msm")},
    "storage_dense_single": {"base": median("dense", "verify", 1, "committer"), "changed": median("dense", "verify", 1, "storage")},
    "storage_dense_concurrent": {"base": median("dense", "verify", 8, "committer"), "changed": median("dense", "verify", 8, "storage")},
    "storage_head_single": {"base": median("head", "verify", 1, "committer"), "changed": median("head", "verify", 1, "storage")},
    "storage_head_concurrent": {"base": median("head", "verify", 8, "committer"), "changed": median("head", "verify", 8, "storage")},
}
(out_path / "summary.json").write_text(json.dumps(summary, indent=2) + "\n")

def percent(pair, field):
    return 100 * (pair["changed"][field] / pair["base"][field] - 1)

def speedup(pair, field="wall_ms"):
    if field == "proofs_per_second":
        return pair["changed"][field] / pair["base"][field]
    return pair["base"][field] / pair["changed"][field]

def row(label, pair, field, unit, digits=1, ratio=False):
    base, changed = pair["base"], pair["changed"]
    delta = f"{speedup(pair, field):.2f}x" if ratio else f"{percent(pair, field):+.1f}%"
    return f"| {label} | {base[field]:.{digits}f} {unit} | {changed[field]:.{digits}f} {unit} | {delta} |"

def headline(label, pair):
    base, changed = pair["base"], pair["changed"]
    return (
        f"| {label} | {base['proofs_per_second']:.2f} witnesses/s | "
        f"{changed['proofs_per_second']:.2f} witnesses/s ({speedup(pair, 'proofs_per_second'):.2f}x) | "
        f"{base['cpu_ms']:.0f} -> {changed['cpu_ms']:.0f} ms ({percent(pair, 'cpu_ms'):+.1f}%) | "
        f"{base['peak_mib']:.0f} -> {changed['peak_mib']:.0f} MiB ({percent(pair, 'peak_mib'):+.1f}%) |"
    )

md = [
    "# Reproduced real-witness results",
    "",
    "Medians of three alternating rounds. In the combined table, one witness/s is one complete witness decode plus proof verification; verification-only tables use verified proofs/s.",
    "",
    "## PRs #2-#5 combined: decode plus proof verification",
    "",
    "| Workload, eight concurrent witnesses | Baseline | After PRs #2-#5 | CPU/witness | Peak memory |",
    "|---|---:|---:|---:|---:|",
    headline("Dense", summary["stack_dense"]),
    headline("Ordinary head", summary["stack_head"]),
    "",
    "## PR #2: witness decoding",
    "",
    "| Dense, one witness | Base | PR #2 | Change |",
    "|---|---:|---:|---:|",
    row("Elapsed", summary["decode_dense"], "wall_ms", "ms", 1, True),
    row("Process CPU", summary["decode_dense"], "cpu_ms", "ms", 0),
    "",
    "## PR #3: proof verification",
    "",
    "| Dense workload | PR #2 | PR #3 | Change |",
    "|---|---:|---:|---:|",
    row("One proof, elapsed", summary["msm_dense_single"], "wall_ms", "ms"),
    row("One proof, process CPU", summary["msm_dense_single"], "cpu_ms", "ms", 0),
    row("One proof, peak memory", summary["msm_dense_single"], "peak_mib", "MiB", 0),
    row("Eight concurrent", summary["msm_dense_concurrent"], "proofs_per_second", "verified proofs/s", 1, True),
    "",
    "## PR #5: proof verification and memory",
    "",
    "| Workload | PR #4 | PR #5 | Change |",
    "|---|---:|---:|---:|",
    row("Dense, one proof", summary["storage_dense_single"], "wall_ms", "ms"),
    row("Dense, one-proof peak", summary["storage_dense_single"], "peak_mib", "MiB", 0),
    row("Dense, eight concurrent", summary["storage_dense_concurrent"], "proofs_per_second", "verified proofs/s", 1, True),
    row("Dense concurrent peak", summary["storage_dense_concurrent"], "peak_mib", "MiB", 0),
    row("Ordinary head, one proof", summary["storage_head_single"], "wall_ms", "ms", 2),
    row("Ordinary-head concurrent peak", summary["storage_head_concurrent"], "peak_mib", "MiB", 1),
    "",
]
(out_path / "SUMMARY.md").write_text("\n".join(md))
print(f"raw rounds: {runs_path}")
print(f"machine-readable medians: {out_path / 'summary.json'}")
print(f"report tables: {out_path / 'SUMMARY.md'}")
PY
```

</details>

<details>
<summary>real-witness-harness.rs</summary>

<!-- file:real-witness-harness.rs -->
```rust
use salt::proof::salt_witness::SaltWitness;
use rayon::prelude::*;
use std::{hint::black_box, time::Instant};
#[repr(C)] struct Timespec { sec: i64, nsec: i64 }
unsafe extern "C" { fn clock_gettime(clock: i32, t: *mut Timespec) -> i32; }
fn cpu() -> f64 { let mut t=Timespec{sec:0,nsec:0}; assert_eq!(unsafe{clock_gettime(2,&mut t)},0); t.sec as f64 + t.nsec as f64*1e-9 }
fn memory() -> serde_json::Value { let s=std::fs::read_to_string("/proc/self/status").unwrap(); let mut m=serde_json::Map::new(); for k in ["VmRSS:","VmHWM:"] {let l=s.lines().find(|l|l.starts_with(k)).unwrap();m.insert(k.trim_end_matches(':').into(),l.split_whitespace().nth(1).unwrap().parse::<u64>().unwrap().into());} m.into() }
fn decode(b:&[u8])->SaltWitness {bincode::serde::decode_from_slice(b,bincode::config::legacy()).unwrap().0}
fn main(){
 let a:Vec<_>=std::env::args().collect(); let dir=&a[1]; let mode=&a[2]; let concurrency:usize=a[3].parse().unwrap(); let repeats:usize=a[4].parse().unwrap();
 let manifest:serde_json::Value=serde_json::from_slice(&std::fs::read(format!("{dir}/manifest.json")).unwrap()).unwrap();
 let files=manifest.as_array().unwrap(); let bytes:Vec<_>=files.iter().map(|f|std::fs::read(format!("{dir}/{}.bin",f["block"].as_u64().unwrap())).unwrap()).collect();
 let witnesses:Vec<SaltWitness>=bytes.iter().map(|b|decode(b)).collect();
 let mut identities=vec![];
 for (i,w) in witnesses.iter().enumerate(){
  w.verify_proof().unwrap();
  let encoded=bincode::serde::encode_to_vec(w,bincode::config::legacy()).unwrap(); assert!(decode(&encoded)==*w, "semantic codec round trip");
  identities.push(serde_json::json!({"block":files[i]["block"],"root":hex::encode(w.state_root().unwrap()),"kvs":w.kvs.len(),"commitments":w.proof.parents_commitments.len(),"salt_wire_hash":blake3::hash(&encoded).to_string(),"salt_wire_bytes":encoded.len()}));
 }
 let prepared_memory=memory();
 let jobs=witnesses.len()*repeats;
 let c=cpu(); let t=Instant::now();
 let work=|i:usize| {let k=i%bytes.len();match mode.as_str(){"verify"=>witnesses[k].verify_proof().unwrap(),"decode"=>{black_box(decode(&bytes[k]));},"full"=>{decode(&bytes[k]).verify_proof().unwrap();},_=>panic!("mode")}};
 if concurrency==1 {for i in 0..jobs {work(i)}} else {for start in (0..jobs).step_by(concurrency){(start..(start+concurrency).min(jobs)).into_par_iter().for_each(work);}}
 let wall=t.elapsed().as_secs_f64(); let process_cpu=cpu()-c;
 println!("{}",serde_json::json!({"mode":mode,"concurrency":concurrency,"rayon_threads":rayon::current_num_threads(),"checks":jobs,"wall_seconds":wall,"cpu_seconds":process_cpu,"wall_ms_per_check":wall*1000./jobs as f64,"cpu_ms_per_check":process_cpu*1000./jobs as f64,"checks_per_second":jobs as f64/wall,"prepared_memory_kib":prepared_memory,"final_memory_kib":memory(),"identities":identities}));
}
```

</details>

