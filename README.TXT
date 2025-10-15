===============================================================
README — Make Total Commander the Default File Manager (Windows)
===============================================================

FILES
-----
1) Set_TotalCommander_Default_All.reg
   • Sets Total Commander (TC) as the default handler for opening folders/drives.
   • Reuses an existing TC window (/O).

2) Restore_Explorer_Default_All.reg
   • Restores Windows Explorer as the default file manager everywhere.

3) Set_TotalCommander_Default_User.reg
   • Per-user variant of #1 (no admin rights required).
   • Changes apply only to the current Windows account.

4) Restore_Explorer_Default_User.reg
   • Per-user variant of #2 (restores Explorer for the current user only).


WHAT EXACTLY GETS CHANGED
-------------------------
We update the shell verbs under these class keys:
  • Directory   → physical filesystem folders (what PowerShell `ii C:\Users\X` usually hits)
  • Folder      → generic shell “Folder” class (virtual folders, some app calls)
  • Drive       → drive roots like C:\, D:\

For each class we:
  • Set the default verb to "open".
  • Set both "open" and "opennewwindow" command lines to TC (or Explorer on restore).
  • Remove (or restore) Explorer-specific COM handlers that override those commands:
      - DelegateExecute
      - DropTarget

This ensures most “open folder” actions route to TC instead of Explorer.


USAGE
-----
A) Make Total Commander the default (system-wide):
   1. Double-click Set_TotalCommander_Default_All.reg
   2. Confirm the UAC prompt.
   3. Restart Windows Explorer: Task Manager → “Windows Explorer” → Restart
   4. Test:
        - WIN+R → type C:\ → Enter (should open in TC)
        - From Chrome/Edge → “Open Downloads folder” (should open in TC)
        - PowerShell → ii C:\  or  ii $env:USERPROFILE

B) Restore Windows Explorer (system-wide):
   1. Double-click Restore_Explorer_Default_All.reg
   2. Restart Windows Explorer (as above).
   3. Test again; folders should open in Explorer.

C) Make or restore defaults per-user (no admin required):
   1. Use Set_TotalCommander_Default_User.reg or Restore_Explorer_Default_User.reg.
   2. These edit only:
        HKEY_CURRENT_USER\Software\Classes
      leaving system-wide HKCR untouched.
   3. No elevation prompt will appear.
   4. Changes take effect after restarting Windows Explorer.

   • Advantages:
       – Safe for standard accounts.
       – Ideal for multi-user machines.
       – Does not require administrator privileges.


SUGGESTED IMPROVEMENTS (Optional)
---------------------------------
1) Fine-tune how TC opens:
   • Reuse existing window (already included): /O
   • Open the target in the LEFT panel:      /L="%1"
   • Open in a new tab:                       /T
   Example command you can drop into the .reg:
     "C:\Program Files\totalcmd\TOTALCMD64.EXE" /O /T /L="%1"

2) Optional “Open in Total Commander” context menu (right-click)
   • Add this as a convenience without changing the system default:
     - For folders:
       [HKCR\Directory\shell\OpenInTotalCommander]
         @="Open in Total Commander"
         "Icon"="C:\\Program Files\\totalcmd\\TOTALCMD64.EXE,0"
       [HKCR\Directory\shell\OpenInTotalCommander\command]
         @="\"C:\\Program Files\\totalcmd\\TOTALCMD64.EXE\" /O \"%1\""
     - Background of a folder (right-click empty space):
       [HKCR\Directory\Background\shell\OpenInTotalCommander]
         @="Open in Total Commander"
         "Icon"="C:\\Program Files\\totalcmd\\TOTALCMD64.EXE,0"
       [HKCR\Directory\Background\shell\OpenInTotalCommander\command]
         @="\"C:\\Program Files\\totalcmd\\TOTALCMD64.EXE\" /O \"%V\""
     - Drives:
       [HKCR\Drive\shell\OpenInTotalCommander]
         @="Open in Total Commander"
         "Icon"="C:\\Program Files\\totalcmd\\TOTALCMD64.EXE,0"
       [HKCR\Drive\shell\OpenInTotalCommander\command]
         @="\"C:\\Program Files\\totalcmd\\TOTALCMD64.EXE\" /O \"%1\""
   • To make it appear only on Shift-right-click, add `"Extended"=""` under each ...\shell\OpenInTotalCommander key.
   • On Windows 11, these appear under “Show more options” (classic menu).


WHY “DelegateExecute” AND “DropTarget” MATTER
---------------------------------------------
• Historically, Explorer opened folders using a simple command line found at:
    ...\shell\open\command  (e.g., explorer.exe "%1")

• Modern Windows also supports COM-based handlers. Two important properties are:
  1) DelegateExecute — a GUID pointing to a COM object (a shell verb handler).
     - If present, Windows **ignores** the plain command line and invokes the COM handler instead.
     - Explorer’s default GUID here is:
         {11dbb47c-a525-400b-9e80-a54615a090c0}
     - That’s why merely changing ...\open\command isn’t enough; Explorer keeps winning.

  2) DropTarget — another COM hook used by Explorer to handle shell verbs and drag-drop.
     - If defined on the verb key, it can similarly bypass your custom command.

• In short: if DelegateExecute (and sometimes DropTarget) exist on "open" or "opennewwindow",
  they take precedence. Removing them (using `"DelegateExecute"=-` and `"DropTarget"=-"`)
  re-enables Windows’ normal command resolution so your ...\open\command to TC is honored.

• When restoring Explorer, we put the default DelegateExecute GUID back so the system behaves
  exactly like a stock Windows install.


LIMITATIONS / EXPECTED BEHAVIOR
-------------------------------
• WIN+E is hard-wired to Explorer in Windows 10/11 and does not follow these overrides.
• Some UWP / Store apps or protected shell locations may still open Explorer.
• Most classic “Open containing folder”, Run dialog (WIN+R → C:\), and PowerShell `ii` calls
  will use TC once the .reg is applied.


TROUBLESHOOTING
---------------
1) “No application is associated…”
   • The default verb may not be set, or open\command is missing.
   • Re-apply Set_TotalCommander_Default_All.reg or Set_TotalCommander_Default_User.reg
     and restart Explorer.

2) Nothing changes after merging
   • Restart Windows Explorer (or sign out/in).
   • Confirm your TC path in the .reg matches your install location.
   • If you used the per-user variant (HKCU\Software\Classes), ensure your session is the one
     testing, and try the HKCR (system-wide) version if needed.

3) Quick checks (PowerShell)
   • These should print "open" and a TC command line:
       (Get-ItemProperty 'Registry::HKEY_CLASSES_ROOT\Directory\shell').'(default)'
       (Get-ItemProperty 'Registry::HKEY_CLASSES_ROOT\Directory\shell\open\command').'(default)'
       (Get-ItemProperty 'Registry::HKEY_CLASSES_ROOT\Drive\shell\open\command').'(default)'
   • Test open actions:
       ii C:\
       ii $env:USERPROFILE
       Start-Process "$env:USERPROFILE"

4) Total Commander option (alternative way)
   • TC → Configuration → Options → Operation → Explorer replacement → “Replace Explorer everywhere”
     (and tick “Reuse existing TC window”).
   • This writes equivalent registry hooks automatically.

5) Restore Explorer if needed
   • Double-click Restore_Explorer_Default_All.reg or Restore_Explorer_Default_User.reg
     → Restart Explorer.


NOTES
-----
• Paths:
  - 64-bit TC default: C:\Program Files\totalcmd\TOTALCMD64.EXE
  - 32-bit TC:         C:\Program Files (x86)\totalcmd\TOTALCMD.exe
  Update the .reg if your path differs.

• Safety:
  - All .reg files only touch the relevant verb keys under Directory/Folder/Drive.
  - The restore files put Explorer’s DelegateExecute GUID back so the system is stock-clean.


VERSION
-------
This README covers Windows 10/11 behavior as of 2025 and should remain valid
unless Microsoft changes shell verb resolution logic or the Explorer DelegateExecute GUID.

---------------------------------------------------------------
Happy navigating with Total Commander!  /O /T /L  are your friends.
---------------------------------------------------------------
