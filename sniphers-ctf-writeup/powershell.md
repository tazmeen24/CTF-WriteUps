# $N1PH€RSxTCTF Write-up: PowerShell

**Challenge:** PowerShell

**Category:** Reverse Engineering

**Flag Format:** `$N1PH€RSxTCTF{x.x.x}`


## Initial Challenge

The challenge provided a Windows command containing a large PowerShell `-encodedcommand`:

```
%COMSPEC% /b /c start /b /min powershell -nop -w hidden -encodedcommand <BASE64>
```

Since PowerShell encoded commands use Base64 + UTF-16LE, I decoded the payload using UTF-16LE. This revealed another PowerShell script instead of the flag.

![Decoding the initial encoded command](./images/01-decode-encodedcommand.png)

![Decoded PowerShell script output](./images/02-decoded-script.png)


## Finding the Next Layer

Inside the decoded script, I found another Base64 payload:

```powershell
[Byte[]]$var_code =
[System.Convert]::FromBase64String('38uqIyMjQ6rGEvFH...')
```

The script then XORed every decoded byte with 35:

```powershell
$var_code[$x] = $var_code[$x] -bxor 35
```

Since `35 = 0x23`, I reproduced this in Python:

```python
decoded = base64.b64decode(inner_b64)
layer2 = bytes(b ^ 0x23 for b in decoded)
```

This produced the next binary layer.

![XOR decoding script in Python](./images/03-xor-decode.png)

## Recognizing the GZIP Layer

There was also a large Base64 blob whose decoded output initially looked like garbage when treated as text.

Instead, I inspected the hexadecimal bytes:

```
1f 8b 08
```

These are the magic bytes for GZIP. So I knew the data was compressed rather than plain text.

After GZIP decompression, the output began with:

```powershell
Set-StrictMode -Version 2
$DoIt = @'
function func_get_proc_address {
```

This revealed another PowerShell layer.

![GZIP magic bytes identified in hex dump](./images/04-gzip-magic-bytes.png)

![Decompressed PowerShell layer](./images/05-decompressed-layer.png)

## Understanding the PowerShell Layer

The script dynamically resolved Windows APIs using `GetProcAddress` and then used `VirtualAlloc` to allocate executable memory.

The important part was:

```
VirtualAlloc
Marshal.Copy
$var_runme.Invoke(...)
```

This indicated that the script was loading and executing raw shellcode in memory.

![VirtualAlloc and Marshal.Copy usage in script](./images/06-virtualalloc-marshalcopy.png)

![Shellcode invocation via $var_runme.Invoke](./images/07-shellcode-invoke.png)

## Analyzing the Shellcode

I inspected the resulting binary with `file`, `xxd`, and `strings`.

The beginning contained raw x86 instructions, confirming that it was shellcode.

The strings also revealed:

```
User-Agent: Mozilla/5.0 ...
149.28.81.19
```

The presence of a User-Agent suggested network communication.

![strings output showing User-Agent and IP address](./images/08-strings-output.png)

I then disassembled the shellcode as 32-bit x86 using:

```
objdump -D -b binary -m i386 -M intel layer2.bin > shellcode.asm
```

The disassembly helped me understand the binary and confirm its network-related functionality. After resolving the API hashes, I identified functions such as `InternetOpenA`, `InternetConnectA`, `HttpOpenRequestA`, `HttpSendRequestA`, `InternetReadFile`, and `VirtualAlloc`.

![Disassembly of shellcode showing resolved WinINet API calls](./images/09-shellcode-disassembly.png)

The flag itself was already visible in the strings output:

```
149.28.81.19
```

I verified that this value was actually embedded in the binary using:

```
grep -aob '149.28.81.19' layer2.bin
```

The corresponding bytes were:

```
31 34 39 2e 32 38 2e 38 31 2e 31 39 00
```

which represent the ASCII string `149.28.81.19`.

![Hex offset verification of the IP string in the binary](./images/10-hex-verification.png)

Therefore, the final flag was ``$N1PH€RSxTCTF{149.28.81.19}``
