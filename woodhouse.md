I've had a great idea for a coding agent / helper.

This project is called Woodhouse and the agent it creates is called Woodhouse.

This project has three major components:
1. VSCode Server
    - This is the vscode webserver that host the inbrowser vscode editor. I plan to either fork or extend the existing docker image for this: lscr.io/linuxserver/code-server
    - This webinterface is the primary interface for the user to interact with the ai
2. VSCode Extension
    - A new custom extension that connects to the users prefferred LLM provider. For MVP I want to just use my own ollama server.
    - This extension should have a normal chat based code editing. I plan to either fork or extend the existing opensource extension Zoo Code: https://github.com/Zoo-Code-Org/Zoo-Code
3. Systemd process
    - An on going process that continues to execute commands even when the web interface is closed.
    - Runs crons, listens to webhooks and websockets, polls github and other external services for events and triggers skills

# Requirements:
The ai chat should feel like the same ai anywhere you go to talk to it. 

The projects also implements a skills repository, personality file, deep memory.

Once per day (schedule can be modified) the agent should read through 
