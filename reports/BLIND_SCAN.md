# Quét độc lập trước khi xem pre-label

Frame: frame_0187.jpg

Số xe nhìn thấy bằng mắt: 26

Hai vị trí dễ bị AI bỏ sót hoặc vẽ sai, kèm mô tả xe:
1. Hai đốm sáng đỏ nhỏ sát nhau trên nền đen — đèn hậu của một xe ở khá xa, rất nhỏ/mờ nên dễ bị model bỏ sót.
2. Một vệt sáng đỏ/cam kéo dài theo chiều ngang, hơi nhòe (motion blur) — đèn phanh/đèn hậu của xe đang chạy, dễ bị nhận nhầm hoặc vẽ lệch box vì viền xe không rõ.

Chạy `python3 tools/lock_blind.py` ngay sau khi điền. Sau đó giữ file này nguyên vẹn.
