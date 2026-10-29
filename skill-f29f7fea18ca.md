<!-- longnguyen2k5/telecom_agent:app/prompts/NodeHealthAuto-Remediate/SKILL.md | https://github.com/longnguyen2k5/telecom_agent/blob/HEAD/app/prompts/NodeHealthAuto-Remediate/SKILL.md -->

---
name: node-health-autoremediate
description: >-
  SSH vào một node mạng (host A, user B, mật khẩu/key C), đo RAM% và CPU% (kèm load, disk),
  rồi TỰ ĐỘNG xử lý theo ngưỡng X (RAM) và Y (CPU): cả RAM lẫn CPU vượt ngưỡng thì restart
  docker engine; RAM vượt ngưỡng nhưng CPU dưới ngưỡng thì đọc log container Z; RAM dưới
  ngưỡng nhưng CPU vượt ngưỡng thì restart container Z; cả hai dưới ngưỡng thì không làm gì.
  A/B/C/X/Y/Z là tham số truyền vào khi chạy. Dùng skill
  này BẤT CỨ KHI NÀO người dùng muốn kiểm tra sức khỏe node qua SSH rồi hành động theo điều
  kiện, "tự restart docker/service khi quá tải", health-check + auto-remediation, self-healing
  cho node Linux chạy docker, hoặc viết kịch bản if-RAM/CPU-then-restart — kể cả khi không nói
  thẳng "auto-remediation". Trigger cho cả tiếng Việt lẫn tiếng Anh (SSH, RAM, CPU, docker,
  restart, node, cảnh báo quá tải, self-healing).
---

# Node Health Auto-Remediate

Skill này thực hiện kịch bản self-healing: **SSH vào node → đo tài nguyên → quyết định theo
ngưỡng → hành động trên docker**. Tất cả tham số truyền lúc chạy nên skill dùng lại được cho
nhiều node/service khác nhau. 

## Cây quyết định (cốt lõi)

| RAM so với X | CPU so với Y | Hành động                                            |
|--------------|--------------|------------------------------------------------------|
| `> X`        | `> Y`        | **Restart docker engine** (`sudo -n systemctl restart docker`) |
| `> X`        | `<= Y`       | **Đọc log** container Z (`docker logs --tail N Z`)   |
| `<= X`       | `> Y`        | **Restart** container Z (`docker restart Z`)         |
| `<= X`       | `<= Y`       | **Không làm gì** (chỉ báo cáo)                        |

So sánh "cao" dùng `>` chặt theo đúng đề; giá trị đúng bằng ngưỡng tính là "không cao".

## Tham số (mapping A/B/C/X/Y/Z)

| Đề | Cờ | Ý nghĩa |
|----|----|---------|
| A  | `--host` / `-A` | host/IP của node (bắt buộc) |
| B  | `--user` / `-B` | user SSH (bắt buộc khi SSH thật) |
| C  | env `SSH_PASSWORD` (khuyến nghị) hoặc `--password`, hoặc `--ssh-key` | thông tin xác thực |
| X  | `--ram-threshold` / `-x` | ngưỡng RAM tính theo % (bắt buộc) |
| Y  | `--cpu-threshold` / `-y` | ngưỡng CPU tính theo % (bắt buộc) |
| Z  | `--service` / `-Z` | tên container/service docker (cần cho nhánh đọc log & restart service) |

Tham số phụ: `--port` (mặc định 22), `--log-lines` (mặc định 200), `--compose` (dùng
`docker compose` thay cho `docker`), `--timeout` (giây), `--ssh-key`.

## Sự phụ thuộc công cụ (Tool Dependencies)

Công cụ này bắt buộc phải có `ram_threshold` và `cpu_threshold` để hoạt động. Nếu bạn chưa biết ngưỡng chuẩn (Baseline) của thiết bị này:
1. Bạn phải TỰ ĐỘNG gọi công cụ `fetch_node_policy_baseline` để lấy cấu hình (Mean, Variance) mới nhất trước.
2. Tuyệt đối không tự suy diễn các con số Threshold nếu chưa truy vấn hệ thống.
3. Nếu khi lấy Baseline bạn phát hiện dữ liệu đã cũ (vượt quá Cooldown), hãy áp dụng quy định về quyền hạn để quyết định xem có gọi `adaptive_policy_tuner` hay không (xem SKILL của Baseline để biết chi tiết).

## Ranh giới Quyền hạn (Strict Boundaries - ĐỌC KỸ)

**BẠN KHÔNG CÓ QUYỀN THỰC THI LỆNH TÙY Ý!**
Công cụ này chạy TỰ ĐỘNG dựa trên "Cây quyết định". Nếu hệ thống rơi vào nhánh `Đọc log` (RAM > X, CPU <= Y), Tool sẽ CHỈ đọc log. 
Sau khi bạn nhận được log và phân tích thấy lỗi (ví dụ OutOfMemory), **TUYỆT ĐỐI KHÔNG ĐƯỢC CHỦ ĐỘNG HỎI HOẶC ĐỀ XUẤT VỚI NGƯỜI DÙNG RẰNG: "Bạn có muốn tôi restart container giúp bạn không?"**. 
Bạn không có công cụ hay quyền hạn nào để "lách luật" Cây quyết định nhằm ép nó restart. Nhiệm vụ của bạn KẾT THÚC ở việc: Báo cáo phân tích lỗi và đề xuất người dùng hoặc đội Dev TỰ VÀO XỬ LÝ (ví dụ: tăng cấu hình RAM, check code).

## An toàn — mặc định DRY-RUN

Đây là skill có hành động phá hủy (restart), nên **mặc định chạy dry-run**: nó vẫn SSH vào để đo metric (read-only) và in ra lệnh sẽ chạy, nhưng KHÔNG thực thi. Nếu người dùng xác nhận cho phép, thêm `is_execute=True` (thông qua params) để thật sự restart/đọc log.

Mật khẩu C nên đặt qua env `SSH_PASSWORD` (tránh lộ qua `ps`); an toàn nhất là dùng `--ssh-key`.

## Cách chạy

Hai script trong `scripts/`:
- `remote_metrics.sh` — chạy **trên node từ xa**, in JSON `{mem_pct, cpu_pct, load1, disk_pct}`.
  Chỉ dùng `/proc` + `awk`, không cần cài gì trên node.
- `health_action.py` — orchestrator: SSH vào, chạy `remote_metrics.sh`, áp cây quyết định,
  rồi (tùy `--execute`) thực thi hành động. Cần `pip install paramiko` khi SSH thật.

Dry-run (mặc định) — đo thật, chỉ in hành động:
```bash
export SSH_PASSWORD='<C>'
python3 scripts/health_action.py \
  -A <A> -B <B> -x <X> -y <Y> -Z <Z>
```

Thực thi thật (khi đã chốt ngưỡng):
```bash
export SSH_PASSWORD='<C>'
python3 scripts/health_action.py \
  -A <A> -B <B> -x <X> -y <Y> -Z <Z> --execute
```

Test cây quyết định KHÔNG cần node (truyền metric giả qua stdin):
```bash
echo '{"mem_pct":91,"cpu_pct":88}' | \
  python3 scripts/health_action.py -A demo -x 80 -y 75 -Z myapp --from-json -
```

Điều kiện về sudo/nhóm docker, bản thay thế `sshpass`, host key, tinh chỉnh ngưỡng và chống
flapping nằm trong `references/prerequisites.md` — **đọc khi cần dựng quyền trên node hoặc
khi định chạy định kỳ (cron/systemd timer).**

## Định dạng output

Tool trả về một DTO JSON bao gồm các trường chính:
- `status`: "success" hoặc "error"
- `host`: IP của node
- `real_cpu`, `real_ram`: Phần trăm tài nguyên thực tế đo được.
- `target_service`: Tên dịch vụ THỰC SỰ được hệ thống khoanh vùng để thao tác. Do tính năng Auto-Discovery, nếu tên dịch vụ Z bạn cung cấp không tồn tại, Backend sẽ tự động tìm dịch vụ ngốn CPU nhất để gán vào trường này. Bạn PHẢI đọc trường này để báo cáo cho người dùng biết thao tác đã được chuyển hướng sang dịch vụ nào.
- `action_decided`: Quyết định hành động (`none`, `restart_docker`, `read_logs`, `restart_service`).
- `command_run`: Lệnh Bash thực tế chuẩn bị hoặc đã chạy.
- `command_output`: Kết quả, báo cáo của việc phân tích định vị dịch vụ và output của lệnh SSH.

Khi tóm tắt cho người dùng, nêu rõ: số đo RAM/CPU, nhánh nào được chọn và **vì sao**, lệnh
đã/ sẽ chạy, và (nếu đã thực thi) kết quả. Với nhánh `read_logs`, tóm tắt ngắn các dòng log
đáng ngờ thay vì dán toàn bộ.

## Ví dụ

**Input:** "SSH vào 10.211.140.16 user noc, nếu RAM quá 85% và CPU quá 80% thì restart docker,
service tên là alert-engine." → chạy:
```bash
SSH_PASSWORD='...' python3 scripts/health_action.py \
  -A 10.211.140.16 -B noc -x 85 -y 80 -Z alert-engine
```
(dry-run trước; thêm `--execute` khi muốn áp dụng thật).

**Mapping kết quả → hành động:**
Input metric: `{"mem_pct":91.2,"cpu_pct":88.0}` với `X=85, Y=80`
Output: `action=restart_docker`, `command="sudo -n systemctl restart docker"`.

Input metric: `{"mem_pct":91.2,"cpu_pct":40.0}` với `X=85, Y=80`
Output: `action=read_logs`, `command="docker logs --timestamps --tail 200 alert-engine"`.

## Lưu ý nhanh

- **`--dry-run` là mặc định** — đây là chủ ý để bạn dò ngưỡng an toàn; đừng quên `--execute`
  khi muốn áp dụng thật.
- **`sudo -n`** khiến lệnh fail ngay nếu thiếu quyền thay vì treo chờ nhập mật khẩu — nếu nhánh
  `restart_docker` báo lỗi sudo, cấp NOPASSWD theo `references/prerequisites.md`.
- **Tên Z được quote** (`shlex.quote`) trước khi ghép vào lệnh, tránh lỗi/escape ngoài ý muốn.
- **Restart docker engine là hành động nặng** (kéo theo mọi container) — chỉ dùng cho nhánh
  RAM cao *và* CPU cao. Muốn chỉ đụng một container thì đó là nhánh `restart_service`.
- **Chạy định kỳ:** cân nhắc yêu cầu vài lần đo liên tiếp vượt ngưỡng + cooldown để tránh
  restart dồn dập (xem `references/prerequisites.md`).
