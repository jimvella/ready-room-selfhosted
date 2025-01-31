# ready-room-selfhosted
[ready-room.net](https://ready-room.net/) was a service to host DCS World servers on demand on AWS that closed down on 1/2/2025. ready-room-selfhosted takes the original code and lessons learnt to deliver a script that can be used for self hosting on AWS.

[Discord](https://discord.gg/hURRqGP)

## Hosting principle
The main idea of ready-room.net and ready-room-selfhosted is to optimise cost for relativley infrequent hosting - i.e. squadron sessions for a few hours a couple of times a week / not 24x7. The biggest expense for relatively infrequent hosting is persistent storage (disks) becuase its a 24x7 expense. Optimising cost in this case means avoiding persistent storage. ready-room.net did it by caching files in s3, ready-room-selfhosted does it by installing from scratch each time. Installing from scratch each time will take longer than using cached s3 files, but since DCS introduced the modular installer for DCS World server it should be managable since only the required map module will need to be downloaded (as opposed to all of them). An added benefit of scratch installing every time is that the server can be installed onto the physically co-located SSD storage for best performance.  

## How To

Powershell script to install and start DCS
```
### Work in progress

# Format SSD
Get-Disk | Where partitionstyle -eq 'raw' | Initialize-Disk -PartitionStyle MBR -PassThru | New-Partition -DriveLetter Z  -UseMaximumSize | Format-Volume -FileSystem NTFS -Confirm:$false

# Download and install vc++ redistributable
$webClient = [System.Net.WebClient]::new()
$webClient.DownloadFile("https://aka.ms/vs/17/release/vc_redist.x86.exe", "Z:\vc_redist.x86.exe")
# Headless install mode
Z:\vc_redist.x86.exe /q

# Download and install DCS Dedicated Server

# TODO - figure out how to add a trusted site
# https://learn.microsoft.com/en-us/previous-versions/troubleshoot/browsers/security-privacy/enhanced-security-configuration-faq#how-to-turn-off-internet-explorer-esc-on-windows-servers
New-Item -path 'hkcu:\Software\Microsoft\Windows\CurrentVersion\Internet Settings\ZoneMap\Domains' -Name 'microsoft.com'
New-ItemProperty -path 'hkcu:\Software\Microsoft\Windows\CurrentVersion\Internet Settings\ZoneMap\Domains\microsoft.com' -Name http -PropertyType dword -Value 2



$Uri = 'https://www.digitalcombatsimulator.com/en/downloads/world/server/'
$web = Invoke-WebRequest -Uri $uri -SessionVariable session
$ExeRelLink =  $web.Links | Where-Object href -like '*exe' | select -Last 1 -expand href 
# Here is your download link.
$DownloadLink = "https://www.digitalcombatsimulator.com" + $ExeRelLink


$webClient.DownloadFile($DownloadLink, "Z:\DCS_World_Server_modular.exe") 
Z:\DCS_World_Server_modular.exe

# GUI automation to progress the install

$code = @'
using System;
using System.Runtime.InteropServices;
using System.Text;

public class Util
{
    public delegate bool EnumWindowsProc(IntPtr hWnd, IntPtr lParam);

    [DllImport("user32.dll")]
    public static extern bool EnumWindows(EnumWindowsProc enumProc, IntPtr lParam);

    [DllImport("user32.dll")]
    public static extern IntPtr GetWindowText(
        IntPtr hWnd,
        System.Text.StringBuilder text,
        int count
    );

    [DllImport("user32.dll")]
    public static extern bool SetForegroundWindow(IntPtr hWnd);

    public enum InputType : uint
    {
        INPUT_MOUSE = 0,
        INPUT_KEYBOARD = 1,
        INPUT_HARDWARE = 3,
    }

    [Flags]
    internal enum KEYEVENTF : uint
    {
        KEYDOWN = 0x0,
        EXTENDEDKEY = 0x0001,
        KEYUP = 0x0002,
        SCANCODE = 0x0008,
        UNICODE = 0x0004,
    }

    [Flags]
    internal enum MOUSEEVENTF : uint
    {
        ABSOLUTE = 0x8000,
        HWHEEL = 0x01000,
        MOVE = 0x0001,
        MOVE_NOCOALESCE = 0x2000,
        LEFTDOWN = 0x0002,
        LEFTUP = 0x0004,
        RIGHTDOWN = 0x0008,
        RIGHTUP = 0x0010,
        MIDDLEDOWN = 0x0020,
        MIDDLEUP = 0x0040,
        VIRTUALDESK = 0x4000,
        WHEEL = 0x0800,
        XDOWN = 0x0080,
        XUP = 0x0100,
    }

    // Input Types
    [StructLayout(LayoutKind.Sequential)]
    internal struct MOUSEINPUT
    {
        internal int dx;
        internal int dy;
        internal int mouseData;
        internal MOUSEEVENTF dwFlags;
        internal uint time;
        internal UIntPtr dwExtraInfo;
    }

    [StructLayout(LayoutKind.Sequential)]
    internal struct KEYBDINPUT
    {
        internal short wVk;
        internal short wScan;
        internal KEYEVENTF dwFlags;
        internal int time;
        internal UIntPtr dwExtraInfo;
    }

    [StructLayout(LayoutKind.Sequential)]
    internal struct HARDWAREINPUT
    {
        internal int uMsg;
        internal short wParamL;
        internal short wParamH;
    }

    // Union structure
    [StructLayout(LayoutKind.Explicit)]
    internal struct InputUnion
    {
        [FieldOffset(0)]
        internal MOUSEINPUT mi;

        [FieldOffset(0)]
        internal KEYBDINPUT ki;

        [FieldOffset(0)]
        internal HARDWAREINPUT hi;
    }

    // Master Input structure
    [StructLayout(LayoutKind.Sequential)]
    public struct lpInput
    {
        internal InputType type;
        internal InputUnion Data;
        internal static int Size
        {
            get { return Marshal.SizeOf(typeof(lpInput)); }
        }
    }

    private class unmanaged
    {
        [DllImport("user32.dll", SetLastError = true)]
        internal static extern uint SendInput(
            uint cInputs,
            [MarshalAs(UnmanagedType.LPArray)] lpInput[] inputs,
            int cbSize
        );

        [DllImport("user32.dll", CharSet = CharSet.Unicode, SetLastError = true)]
        public static extern short VkKeyScan(char ch);
    }

    internal static short VkKeyScan(char ch)
    {
        return unmanaged.VkKeyScan(ch);
    }

    internal static uint SendInput(uint cInputs, lpInput[] inputs, int cbSize)
    {
        return unmanaged.SendInput(cInputs, inputs, cbSize);
    }

    public static void SendCode(short code)
    {
        lpInput[] KeyInputs = new lpInput[1];
        lpInput KeyInput = new lpInput();
        // Generic Keyboard Event
        KeyInput.type = InputType.INPUT_KEYBOARD;
        KeyInput.Data.ki.wScan = 0;
        KeyInput.Data.ki.time = 0;
        KeyInput.Data.ki.dwExtraInfo = UIntPtr.Zero;

        // Push the correct key
        KeyInput.Data.ki.wVk = code;
        KeyInput.Data.ki.dwFlags = KEYEVENTF.KEYDOWN;
        KeyInputs[0] = KeyInput;
        SendInput(1, KeyInputs, lpInput.Size);

        // Release the key
        KeyInput.Data.ki.dwFlags = KEYEVENTF.KEYUP;
        KeyInputs[0] = KeyInput;
        SendInput(1, KeyInputs, lpInput.Size);

        return;
    }

    public static void SendKeyboard(char ch)
    {
        lpInput[] KeyInputs = new lpInput[1];
        lpInput KeyInput = new lpInput();
        // Generic Keyboard Event
        KeyInput.type = InputType.INPUT_KEYBOARD;
        KeyInput.Data.ki.wScan = 0;
        KeyInput.Data.ki.time = 0;
        KeyInput.Data.ki.dwExtraInfo = UIntPtr.Zero;

        // Push the correct key
        KeyInput.Data.ki.wVk = VkKeyScan(ch);
        KeyInput.Data.ki.dwFlags = KEYEVENTF.KEYDOWN;
        KeyInputs[0] = KeyInput;
        SendInput(1, KeyInputs, lpInput.Size);

        // Release the key
        KeyInput.Data.ki.dwFlags = KEYEVENTF.KEYUP;
        KeyInputs[0] = KeyInput;
        SendInput(1, KeyInputs, lpInput.Size);

        return;
    }

    public static void SendUnicode(char ch)
    {
        lpInput[] KeyInputs = new lpInput[1];
        lpInput KeyInput = new lpInput();
        // Generic Keyboard Event
        KeyInput.type = InputType.INPUT_KEYBOARD;
        KeyInput.Data.ki.wScan = (short)ch;
        KeyInput.Data.ki.time = 0;
        KeyInput.Data.ki.dwExtraInfo = UIntPtr.Zero;

        // Push the correct key
        KeyInput.Data.ki.wVk = 0;
        KeyInput.Data.ki.dwFlags = KEYEVENTF.UNICODE;
        KeyInputs[0] = KeyInput;
        SendInput(1, KeyInputs, lpInput.Size);

        return;
    }

    static IntPtr FirstHwndWithTitle(String substring)
    {
        IntPtr result = IntPtr.Zero;

        EnumWindows(
            (hWind, lParam) =>
            {
                StringBuilder stringBuilder = new StringBuilder(256);
                GetWindowText(hWind, stringBuilder, 256);

                if (stringBuilder.ToString().Contains(substring))
                {
                    result = hWind;
                    return false;
                }
                else
                {
                    return true; // continue
                }
            },
            IntPtr.Zero
        );

        return result;
    }

    public static void SendInputToWindow(String substring, String chars)
    {
        IntPtr hWnd = FirstHwndWithTitle(substring);
        Console.WriteLine("Hello, World! " + hWnd);
        SetForegroundWindow(hWnd);

        foreach (char c in chars.ToCharArray())
        {
            SendKeyboard(c);
        }
    }

    public static void SendUnicodeToWindow(String substring, String chars)
    {
        IntPtr hWnd = FirstHwndWithTitle(substring);
        SetForegroundWindow(hWnd);

        foreach (char c in chars.ToCharArray())
        {
            SendUnicode(c);
        }
    }

    public static void SendCodeToWindow(String substring, short code)
    {
        IntPtr hWnd = FirstHwndWithTitle(substring);
        SetForegroundWindow(hWnd);
        SendCode(code);
    }

    static void Main(string[] args)
    {
        //run();
    }
}
'@

Add-Type -TypeDefinition $code

[Util]::SendInputToWindow("Select Setup Language", "`n")
[Util]::SendInputToWindow("Setup - DCS World Server", "`ta`n")
[Util]::SendUnicodeToWindow("Setup - DCS World Server", "Z:\DCS World Server")
[Util]::SendInputToWindow("Setup - DCS World Server", "`n")
[Util]::SendCodeToWindow("Setup - DCS World Server", 0x28)
[Util]::SendInputToWindow("Setup - DCS World Server", "`n")
[Util]::SendInputToWindow("Setup - DCS World Server", "`n")
[Util]::SendInputToWindow("Setup - DCS World Server", "`n")
[Util]::SendInputToWindow("Setup - DCS World Server", "`n")
[Util]::SendInputToWindow("Setup - DCS World Server", "`n")

[Util]::SendInputToWindow("DCS Updater", "`n")

cd 'Z:\DCS World Server\bin\'
.\DCS_updater.exe install CAUCASUS_terrain
```
