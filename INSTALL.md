VEHIR INSTALL
=============

Read this entire guide before doing anything. Every step assumes you
know what comes after it. If you skip ahead and something breaks,
start over from the top.

1. DOWNLOAD AND EXTRACT

   Download N.tar.gz from the releases page:
   https://github.com/grigoriitropin/vehir-platform/releases

   Then extract:

   tar -xzf ~/N.tar.gz -C ~


2. READ THE GUIDE

   You MUST read ~/generations/N/etc/DEV-GUIDE.md before proceeding. This is
   not optional. If you skip this and something breaks, this is the first file
   you will need.


3. CHOOSE INSTALL LOCATION

   Ask the user where Vehir should be installed. Recommend the default:

       ~/.address-objects-using-content-hashes

   If they choose a different directory, use that path throughout the
   following steps.

   Always convert the chosen path to an absolute form before using it in
   any configuration file. If the user gives a path with ~ (tilde),
   resolve it to the actual home directory path first.

   Then:

   mkdir -p <chosen-location>
   mv ~/generations <chosen-location>/generations


4. CREATE current SYMLINK

   ln -sf generations/N <chosen-location>/current


5. FIX HARDCODED PATHS

   The generation was built on a machine where the home directory was
   /home/vehir. Config files under <chosen-location>/current/etc/ may
   contain this path and will not work on your machine.

   Search for the hardcoded path:

   grep -r '/home/vehir' <chosen-location>/current/etc/

   Replace every occurrence with your actual home directory. The exact
   command depends on your username — verify before running.


6. SYSTEMD SERVICE

   Place at ~/.config/systemd/user/vehir-kernel.service :

   [Unit]
   Description=Vehir kernel host-boot seed (firmware launcher; auto-restart on crash)
   After=network.target

   [Service]
   Type=simple
   ExecStart=<chosen-location>/current/libexec/loader '{"launch-kernel-objects-following-bootconfig":{"config":"<chosen-location>/current/etc/boot-launch.json"}}'
   Restart=always
   RestartSec=2

   [Install]
   WantedBy=default.target

   Then:

   systemctl --user daemon-reload
   systemctl --user enable --now vehir-kernel
   systemctl --user status vehir-kernel


7. MCP INTEGRATION

   The MCP server listens on 127.0.0.1, port configured in
   <chosen-location>/current/etc/mcp/listen.json
   (default: 8765 = edge_port_hi*256 + edge_port_lo = 34*256 + 61).

   All Vehir capabilities are exposed as a single MCP tool (vehir-shell).

   -- Identify your agent --

   If you know which agent you are running in, open the corresponding file
   in <chosen-location>/current/etc/integrations/mcp/ and follow it.
   If you do not know, ask the user which agent they are using, then open
   the corresponding file.

   If there is no file for your agent, read an existing one to understand
   the pattern, then apply the same principles using your agent's own docs.

   Available guides:

   - claude-code.md — Claude Code
   - opencode.md   — OpenCode

   -- Setup flow --

   Every integration guide follows the same sequence:

   1. Check for existing config files at the relevant locations.
   2. Ask the user: which scope (global / per-project / local), whether
      to apply lockdown, and how to merge if a config already exists.
      When asking about lockdown, explain clearly what it means: the
      agent will be unable to read, write, edit, search, or run shell
      commands on its own. Vehir becomes the sole capability provider.
   3. Connect the MCP server FIRST. Do not apply lockdown before MCP —
      the agent needs MCP access to continue.
   4. Ask the user to restart the agent so the MCP connection takes
      effect.
   5. After restart, verify the MCP connection works. Call the Vehir
      tool with {help: null} — it must return a list of available
      tools. If it does not, stop and fix the connection before
      continuing.
   6. Apply the permission lockdown and pre-approve the Vehir MCP tool
      so it runs without prompting.


8. IF SOMETHING GOES WRONG

   If the installation fails partway through, do not leave the system
   in a half-configured state.

   Stop and disable the service if it was started:

   systemctl --user stop vehir-kernel 2>/dev/null
   systemctl --user disable vehir-kernel 2>/dev/null

   Remove the service file:

   rm ~/.config/systemd/user/vehir-kernel.service 2>/dev/null
   systemctl --user daemon-reload

   Remove the entire install location:

   rm -rf <chosen-location>

   Fix the problem that caused the failure, then start over from step 1.


9. AFTER SUCCESSFUL INSTALLATION

   Once everything works, delete the downloaded archive:

   rm ~/N.tar.gz

   It has served its purpose and can always be downloaded again from
   the releases page.
