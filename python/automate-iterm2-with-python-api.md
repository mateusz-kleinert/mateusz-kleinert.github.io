# Automate iTerm2 with Python API

[iTerm2](https://iterm2.com/) can be automated and extended using a built‑in Python API that exposes windows, tabs, panes, profiles, and more through the iterm2 module. This API lets you script common tasks like project setup, layout management, or environment bootstrapping directly from Python.​

# What is iTerm2?

iTerm2 is a feature‑rich terminal emulator for macOS that adds advanced capabilities on top of the standard Terminal.app, including split panes, profiles, triggers, and deep customization. For automation, it offers a scripting system that can be driven via Python, enabling tight integration with workflows and development tools.​

# iTerm2 Python API

The iTerm2 Python API is exposed as the iterm2 Python package, which talks to a running iTerm2 instance using protocols like Google Protocol Buffers and websockets, abstracted away behind a high‑level interface. Scripts are typically asynchronous: you define an async main function that receives a connection object and then operate on the app, windows, tabs, and sessions through awaitable methods.​

# Enabling and running scripts

To use the Python API, you must enable the Python API server in iTerm2’s preferences under the general settings (often labeled “Enable Python API server”). Scripts are usually stored under `~/Library/ApplicationSupport/iTerm2/Scripts` and can be run from the Scripts menu, from the command line with helper launchers, or automatically on startup via the `AutoLaunch` directory (`Applescript` files).​

# Example

In this example we will open a new tab with a one wide panel at the top and two panels of the same size at the bottom. It should show an example of the layout bootstraping. In addition we will launch sample applications, `htop` to monitor CPU, RAM and processes at the top, `iotop` to monitor real-time per-process disk I/O at the bottom-left panel and [trippy](https://trippy.rs/) for real-time network monitoring.

```python
import iterm2
import time

async def main(connection):
    app = await iterm2.async_get_app(connection)
    window = app.current_terminal_window

    if window is not None:
        # Create a new tab
        tab = await window.async_create_tab()
        
        # Store a reference to the top panel
        top_panel = tab.current_session

        # Execute command in the top panel
        await top_panel.async_send_text('htop\n')

        # Create a bottom panel by splitting the top one horizontally
        bottom_left_panel = await top_panel.async_split_pane(vertical=False)
        await bottom_left_panel.async_send_text('sudo iotop\n')

        # Split the bottom panel into left and right panels
        bottom_right_panel = await bottom_left_panel.async_split_pane(vertical=True)
        await bottom_right_panel.async_send_text('trip -u localhost\n')
    else:
        print("No window available")

iterm2.run_until_complete(main)
```
