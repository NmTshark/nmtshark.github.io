---
author: NmToan
date: 2025-12-29T04:59:04.866Z
lastmod: 2025-12-29T13:39:20.763Z
title: WOW!
slug:  
featured: false
draft: false
tags:
  - Forensics
  
description:  |
  
---
# Write-up: `report.pdf` -> `Y.png`

Muc tieu cua tai lieu nay la ghi lai day du toan bo qua trinh phan tich tu luc nhan file `report.pdf` cho den khi giai ma ra duoc anh SSTV va tach cac kenh mau, trong do co file `Y.png`.

Tai lieu nay dung dung cac file da tao trong workspace de ban co the tiep tuc phan tich tu moc `Y.png` ma khong can lam lai tu dau.

## 1. Tong quan challenge

Ban dau chi co file:

- `report.pdf`

Noi dung trong PDF noi ve:

- co mot ban tin vo tuyen bat thu duoc trong sa mac
- co cau "I will transfer the data utilizing Papa Delta"
- co mo ta vat the "something similar to the galaxy"

Ngay tu phan text da co hai huong quan trong:

- co du lieu am thanh/an tin hieu duoc dinh kem trong PDF
- "Papa Delta" rat giong cach goi mode `PD` trong SSTV

## 2. Kiem tra file ban dau

Buoc dau tien la liet ke file va xac nhan workspace:

```powershell
Get-ChildItem -Force
rg --files
```

Sau do kiem tra nhanh thuoc tinh file:

```powershell
Get-Item report.pdf | Format-List *
```

Ket qua quan trong:

- `report.pdf` co kich thuoc khoang `5.2 MB`
- kich thuoc lon hon rat nhieu so voi mot PDF text thong thuong
- kha nang cao co object nhung lon ben trong PDF

## 3. Doc text thuan va object cua PDF

Do may khong co day du cac tool Linux quen thuoc nhu `pdfinfo`, `qpdf`, `exiftool`, minh chuyen sang doc truc tiep bang `strings` va Python.

Lenh thu text thuan:

```powershell
strings -n 6 report.pdf | Select-Object -First 300
```

Trong output co cac object quan trong:

- object page
- object content
- va dac biet la mot object:

```text
/Type /Sound /R 48000 /C 1 /B 16 /E /muLaw
/Length 5155787
/Filter /FlateDecode
```

Day la dau hieu manh nhat cho thay:

- PDF co nhung mot `Sound object`
- stream am thanh da bi nen bang `FlateDecode`
- payload am thanh rat lon, hon `5 MB`

## 4. Liet ke cac object stream trong PDF bang Python

De chac chan hon, minh viet mot doan Python nho:

- quet tung object trong PDF
- tim object co `stream`
- neu co `FlateDecode` thi giai nen
- in ra preview

Y tuong cua script:

```python
import re, zlib, pathlib
p = pathlib.Path('report.pdf').read_bytes()
for m in re.finditer(rb'(\d+)\s+(\d+)\s+obj(.*?)endobj', p, re.S):
    ...
```

Ket qua quan trong nhat:

- `OBJ 25` la `Type /Sound`
- stream raw dai `5155788` bytes
- sau khi `zlib.decompress()` thi stream dai `56203108` bytes
- preview cua stream sau giai nen bat dau bang:

```text
RIFF....WAVEfmt ...
```

Tuc la:

- stream giai nen thanh mot file `WAV` that
- du lieu can phan tich chinh la am thanh nhung trong PDF

## 5. Trich xuat object am thanh ra file `extracted.wav`

Sau khi xac nhan object `25` la audio, minh trich rieng no ra:

```python
import re, zlib, pathlib
p = pathlib.Path('report.pdf').read_bytes()
m = re.search(rb'25\s+0\s+obj(.*?)stream\r?\n', p, re.S)
start = m.end()
end = p.index(b'endstream', start)
stream = p[start:end]
out = zlib.decompress(stream)
pathlib.Path('extracted.wav').write_bytes(out)
```

Ket qua:

- tao ra file `extracted.wav`
- kich thuoc khoang `56 MB`

File duoc tao ra trong workspace:

- `extracted.wav`

## 6. Kiem tra header va cau truc cua WAV

Ban dau co ve nhu file la:

- stereo
- 16-bit
- 48000 Hz

nhung khi doc ky hon thi thay header hoi "la".

Minh dung Python kiem tra:

```python
import pathlib, struct
p = pathlib.Path('extracted.wav').read_bytes()
print(p[:64])
print(p.find(b'fmt '))
print(p.find(b'data'))
```

Quan sat quan trong:

- file bat dau bang `RIFF ... WAVE`
- co `fmt ` chunk
- co `data` chunk
- nhung cach dien giai theo `wave` cho ket qua khong on dinh

Minh tiep tuc doc payload `data` truc tiep thay vi phu thuoc vao parser `wave`.

## 7. Phat hien header dang "giau" mono 32-bit

Khi doc `data` chunk theo `int16`, kieu du lieu trong hai kenh nhin rat ky:

- kenh trai co bien do thay doi that
- kenh phai gan nhu chi toan `0` hoac `-1`

Luc do minh thu doc cung payload bang `int32`:

```python
np.frombuffer(raw, dtype='<i4')
```

Ket qua hop ly hon nhieu.

Suy ra:

- header WAV duoc sap dat de trong giong `stereo 16-bit`
- nhung payload thuc te hop ly hon khi xem la `mono 32-bit PCM`

Day la mot phat hien rat quan trong, vi no anh huong truc tiep toi viec giai dieu che SSTV.

## 8. Phan tich pho tan so cua `extracted.wav`

De xem audio la gi, minh dung `matplotlib` ve spectrogram:

```python
ax.specgram(sig, NFFT=4096, Fs=48000, noverlap=3072, cmap='magma')
```

Sinh ra:

- `spectrogram_full.png`
- `spectrogram_full2.png`

Quan sat:

- co cac vach lap lai rat deu
- day la tin hieu dieu che, khong phai nhac hay voice thong thuong
- thoi luong tong the khoang `292.7 s`

Sau do minh tinh khoang cach giua cac sync pulse va thay:

- pulse lap lai khoang `44993` samples

Minh so sanh voi cac mode PD:

- `PD240` neu sample rate la `44993 Hz`
- `PD290` neu sample rate la `~48000 Hz`

Do sample rate logic cua file la `48000`, mode hop ly nhat la:

- `PD290`

## 9. Vi sao xac dinh mode la `PD290`

Minh dung cong thuc scanline cua PD mode:

```text
scanline = 0.020 + 0.00208 + 4 * channelSeconds
```

So sanh cac mode:

- `PD50`
- `PD90`
- `PD120`
- `PD160`
- `PD180`
- `PD240`
- `PD290`

Khoang cach pulse do duoc la `44993 samples`.

Tinh nguoc ra:

- neu mode la `PD290` thi sample rate xap xi `48003.8 Hz`
- day gan nhu khop hoan hao voi `48000 Hz`

Them vao do:

- so pulse tim duoc la `308`
- `PD290` co anh kich thuoc `800 x 616`
- mode nay dung `308` scanline, moi scanline sinh ra 2 dong anh

Suy ra rat chac:

- day la SSTV `PD290`

## 10. Doi chieu voi implementation goc trong repo `robot36`

De tranh giai tay sai cong thuc, minh clone va doc repo tham khao:

- `_tmp_robot36`
- `_tmp_PicoSSTV`
- `_tmp_sstv_decoder`

Trong `_tmp_robot36`, file quan trong nhat la:

- `_tmp_robot36/app/src/main/java/xdsopl/robot36/PaulDon.java`
- `_tmp_robot36/app/src/main/java/xdsopl/robot36/Decoder.java`
- `_tmp_robot36/app/src/main/java/xdsopl/robot36/Demodulator.java`

Tu `Decoder.java`, minh rut ra duoc:

- VIS code `94` ung voi `PD290`
- kich thuoc anh `800 x 616`
- `channelSeconds = 0.2288`

Tu `PaulDon.java`, minh rut ra bo cuc scanline:

- `sync pulse`: `20 ms`
- `sync porch`: `2.08 ms`
- 4 phan lien tiep:
  - `Y even`
  - `V average`
  - `U average`
  - `Y odd`

Offset tinh theo sample:

- `begin = round(0.00208 * fs)`
- `v_avg = round((0.00208 + 0.2288) * fs)`
- `u_avg = round((0.00208 + 0.2288 * 2) * fs)`
- `y_odd = round((0.00208 + 0.2288 * 3) * fs)`

## 11. Demod tin hieu theo huong giong `robot36`

Minh tu viet decoder Python bat chuoc huong xu ly trong `robot36`:

1. Doc `mono 32-bit` tu `extracted.wav`
2. Mix xuong baseband quanh `1900 Hz`
3. Low-pass quanh bang thong SSTV
4. FM demod de dua ve tan so chuan hoa
5. Tim sync pulse `1200 Hz`
6. Tinh chieu dai scanline
7. Lay cac mau theo bo cuc `Y / V / U / Y`
8. Chuyen `YUV -> RGB`

Phan FM demod duoc mo phong bang:

```python
freq[1:] = (fs / (bw*np.pi)) * np.angle(base[1:] * np.conj(base[:-1]))
```

Sau do minh smooth tin hieu va tim pulse:

```python
mask = smooth < -1.5
```

## 12. Dung lai anh SSTV dau tien

Sau khi map theo bo cuc `PD290`, minh dung duoc anh:

- `pd290_decoded.png`

Sau do thu them phien ban lam min giong EMA trong `robot36`:

- `pd290_decoded_ema.png`

Cuoi cung, sau khi hieu ro hon ve payload `mono 32-bit`, minh tao lai anh tot nhat:

- `pd290_mono32.png`

Anh nay la moc quan trong nhat cua qua trinh giai ma audio -> image.

## 13. Hinh dang cua `pd290_mono32.png`

Anh thu duoc co dac diem:

- nen sang
- mot khoi den xep thanh dang xoan oc
- rat giong hinh thien ha xoan / whirlpool / galaxy stylized

Day phu hop voi text trong PDF:

```text
something similar to the galaxy
```

Dong thoi, no xac nhan huong SSTV `PD` la dung.

## 14. Kiem tra lai moc sync va VIS header

Sau nay minh con kiem tra ky hon:

- tim theo `start of sync pulse`
- thay vi chi dua vao `end of sync pulse`

Anh tao lai:

- `pd290_startsync.png`

Quan sat:

- ve tong the van ra cung mot hinh xoan oc
- khong lam thay doi ban chat ket qua: audio duoc giai dung thanh mot anh dang galaxy/spiral

Minh cung doc duoc VIS header o dau file va thay:

- cac bit phu hop voi mode `PD290`

Nen ket luan den day van giu nguyen:

- am thanh trong PDF la SSTV `PD290`

## 15. Phan tich sau anh `pd290_mono32.png`

Sau khi da co anh, minh bat dau mo rong theo nhieu huong:

- xoay anh
- flip
- log-polar
- polar
- untwirl
- dechirp
- FFT
- zoom center
- tim QR
- tach bit plane
- tach kenh mau

Nhung theo yeu cau cua ban, tai lieu nay se dung chi tiet den moc tim ra `Y.png`.

## 16. Tach cac kenh mau RGB

Tu `pd290_mono32.png`, minh tach 3 kenh mau:

```python
img = np.array(Image.open('pd290_mono32.png').convert('RGB'))
Image.fromarray(img[:,:,0]).save('chan_R.png')
Image.fromarray(img[:,:,1]).save('chan_G.png')
Image.fromarray(img[:,:,2]).save('chan_B.png')
```

Sinh ra:

- `chan_R.png`
- `chan_G.png`
- `chan_B.png`

Y nghia:

- de xem thong tin co nam rieng tren mot kenh khong
- de phuc vu cac phep bien doi tiep theo

## 17. Tao bit-planes cho tung kenh

Minh tiep tuc tach bit-plane de tim watermark/stego:

```python
for c,name in enumerate('RGB'):
    for b in range(8):
        plane=((img[:,:,c]>>b)&1)*255
```

Sinh ra:

- `bit_R0.png` ... `bit_R7.png`
- `bit_G0.png` ... `bit_G7.png`
- `bit_B0.png` ... `bit_B7.png`

Buoc nay huu ich de:

- xem co text/QR trong LSB/MSB khong
- kiem tra stego muc do co ban

## 18. Tao cac anh difference giua cac kenh

De lam ro thong tin an trong do lech mau, minh tao them:

- `RminusG.png`
- `BminusG.png`
- `RminusB.png`
- `GminusRB.png`
- `sum.png`

Y tuong:

- neu chu/hoa tiet duoc nhung nhat trong kenh U/V hoac trong sai khac giua cac kenh, anh difference se lam no noi len

## 19. Tinh xap xi kenh YUV tu RGB

Sau do minh quy doi RGB -> YUV gan dung:

```python
Y = 0.299*R + 0.587*G + 0.114*B
U = -0.14713*R - 0.28886*G + 0.436*B
V = 0.615*R - 0.51499*G - 0.10001*B
```

Tu day sinh ra 3 file:

- `Y.png`
- `U.png`
- `V.png`

Day la moc ma ban muon dung lai de tu phan tich tiep.

## 20. Y nghia cua `Y.png`

`Y.png` la kenh do sang xap xi cua anh `pd290_mono32.png`.

No huu ich vi:

- neu thong tin an chu yeu nam trong do sang thi `Y.png` se giu no ro nhat
- loai bo phan nao anh huong cua sai khac mau
- phu hop de tiep tuc cac phep OCR, threshold, morph, polar unwrap, FFT, hoac stego visual

Trong workspace hien co:

- `Y.png`
- `U.png`
- `V.png`

va ban co the tiep tuc phan tich tu day.

## 21. Danh sach cac file chinh da tao theo thu tu logic

Theo thu tu quan trong cua pipeline, cac file can quan tam nhat la:

1. `report.pdf`
2. `extracted.wav`
3. `spectrogram_full.png`
4. `spectrogram_full2.png`
5. `pd290_decoded.png`
6. `pd290_decoded_ema.png`
7. `pd290_mono32.png`
8. `chan_R.png`
9. `chan_G.png`
10. `chan_B.png`
11. `bit_R0.png` ... `bit_B7.png`
12. `RminusG.png`
13. `BminusG.png`
14. `RminusB.png`
15. `GminusRB.png`
16. `sum.png`
17. `Y.png`
18. `U.png`
19. `V.png`

## 22. Tom tat ky thuat

Toan bo luong xu ly tu A -> Z den `Y.png` la:

1. Xac nhan `report.pdf` co kich thuoc bat thuong.
2. Doc object trong PDF va phat hien `Type /Sound`.
3. Giai nen stream am thanh tu object `25`.
4. Xuat stream thanh `extracted.wav`.
5. Phat hien payload hop ly hon khi xem la `mono 32-bit`.
6. Ve spectrogram va nhan ra tin hieu SSTV.
7. So sanh scanline voi cac mode `PD`.
8. Ket luan mode la `PD290`.
9. Doi chieu cong thuc voi implementation `robot36`.
10. Tu viet decoder Python theo bo cuc `Y/V/U/Y`.
11. Dung anh `pd290_decoded.png`.
12. Tinh chinh smoothing va cach doc payload, dung anh `pd290_mono32.png`.
13. Tach RGB, bit-plane, anh difference.
14. Quy doi sang xap xi `Y/U/V`.
15. Sinh ra `Y.png`, `U.png`, `V.png`.

## 23. Ghi chu cho lan phan tich tiep theo

Neu tiep tuc tu `Y.png`, cac huong hop ly nhat la:

- threshold nhieu muc
- morphology open/close
- polar hoac log-polar unwrap
- OCR tren cac crop sau threshold
- contour analysis
- local contrast enhancement
- FFT / autocorrelation
- skeletonization neu nghi la text bi warp

Nhung trong tai lieu nay, minh dung lai tai moc:

- `Y.png`

de ban tiep tuc phan tich.
