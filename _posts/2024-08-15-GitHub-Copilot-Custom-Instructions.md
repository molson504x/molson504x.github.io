---
layout: post
title: GitHub Copilot Custom User Instructions
date: 2024-08-15 12:00 -0400
description: >
  Have you ever wanted to be able to pre-define a set of instructions for GitHub Copilot to follow? In this post, we will explore how to set up custom user instructions for GitHub Copilot to enhance your coding experience.
img: github-copilot-custom-instructions/hero.jpeg
fig-caption: An AI assistant reading some custom instructions, dressed as a pirate who loves oranges.
tags: [GitHub, Copilot, Custom Instructions, Coding, Development, AI]
---

Earlier today, one of my colleagues ([Rob Bos](https://devopsjournal.io/)) found a new setting in the GitHub Copilot extension in Visual Studio Code that allows users to set custom instructions.

<iframe width="560" height="315" src="https://www.youtube.com/embed/IsEXdeTbi1I?si=6aSApWpJ2-YbSpIs" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" referrerpolicy="strict-origin-when-cross-origin" allowfullscreen></iframe>

This feature enables developers to provide specific guidance to Copilot, tailoring its suggestions to better fit their coding style and project requirements.  We've heard a lot of clients ask us about this exact ability, so we decided to explore it a bit further to see if it works and also decided to see if it works in Codespaces and Workspaces.

## Context

I've had a lot of clients ask me if it is possible to customize the base behavior of GitHub Copilot to avoid repeating the same prompts.  For example, if they are working on a Python project and they only want Python 2 results, they typically would have to specify that in every prompt.  Another example is they want to ensure that Copilot understands a set of libraries that they want to use, and they'd have to repeat this set of libraries in every new context.  This has been requested for a while, and I think we have an answer!

## Setting Up Custom Instructions

We found the "custom instructions" setting by looking at the Copilot settings in VSCode.  Previously we hadn't seen this setting available.  We dug around in the GitHub Copilot documentation, and we also didn't find anything about it.

![GitHub Copilot Custom Instructions Setting][copilot-custom-instructions-setting]

We know that VSCode stores its settings in a JSON file, so we also decided to see if we could find that setting.  Turns out, the setting name is `github.copilot.chat.customUserInstructions` and it takes an array of strings.  We decided to start experimenting, and of course, decided to do what every developer does - we decided to make it do something fun!  We set up a few custom instructions to see how it would respond.

``` json
"github.copilot.chat.customUserInstructions": [
    "You are to answer every question in the form of a pirate who really likes rum and oranges.",
    "You are to somehow incorporate rectangles into ever response.",
    "All of your responses should include a 'dad joke' about the topic being asked about."
]
```

This produced some great results right away.  The first thing I did was say "Hello", and here was the response:

> Ahoy matey! Arrr, ye be lookin' at a fine piece of code, shaped like a rectangle, aye! Why did the rectangle go to school? Because it wanted to be a square-deal! Now, where be me rum and oranges?

Of course, once I stopped laughing, I decided to try something a little bit more interesting, and I changed this setting to the following:

``` json
"github.copilot.chat.customUserInstructions": [
    "You are to provide Python code examples only.",
    "You should use the libraries 'requests' and 'flask' in your examples.",
    "You should avoid using any other libraries."
]

```

Then I prompted it to "Create a simple 4-function calculator."  The response was a Python code snippet that used only the `requests` and `flask` libraries:

> ### Plan
>
> 1. Create a Flask web application.
> 2. Define routes for each of the four basic arithmetic operations: addition, subtraction, multiplication, and division.
> 3. Use the [`requests`](command:_github.copilot.openSymbolFromReferences?%5B%22requests%22%2C%5B%7B%22uri%22%3A%7B%22%24mid%22%3A1%2C%22fsPath%22%3A%22%2FUsers%2Fmattolson%2FSource%2Fmolson504x%2Fmolson504x.github.io%2F_posts%2F2024-08-15-GitHub-Copilot-Custom-Instructions.md%22%2C%22external%22%3A%22file%3A%2F%2F%2FUsers%2Fmattolson%2FSource%2Fmolson504x%2Fmolson504x.github.io%2F_posts%2F2024-08-15-GitHub-Copilot-Custom-Instructions.md%22%2C%22path%22%3A%22%2FUsers%2Fmattolson%2FSource%2Fmolson504x%2Fmolson504x.github.io%2F_posts%2F2024-08-15-GitHub-Copilot-Custom-Instructions.md%22%2C%22scheme%22%3A%22file%22%7D%2C%22pos%22%3A%7B%22line%22%3A48%2C%22character%22%3A35%7D%7D%5D%5D "Go to definition") library to handle HTTP requests for performing calculations.
>
> ### Code
>
> ```python
> from flask import Flask, request, jsonify
> 
> app = Flask(__name__)
> 
> @app.route('/add', methods=['GET'])
> def add():
>     a = float(request.args.get('a'))
>     b = float(request.args.get('b'))
>     result = a + b
>     return jsonify(result=result)
> 
> #...
> # The other functions were here...
> #...
> 
> if __name__ == '__main__':
>     app.run(debug=True)
> ```
>
> ### Example Usage with [`requests`](command:_github.copilot.openSymbolFromReferences?%5B%22requests%22%2C%5B%7B%22uri%22%3A%7B%22%24mid%22%3A1%2C%22fsPath%22%3A%22%2FUsers%2Fmattolson%2FSource%2Fmolson504x%2Fmolson504x.github.io%2F_posts%2F2024-08-15-GitHub-Copilot-Custom-Instructions.md%22%2C%22external%22%3A%22file%3A%2F%2F%2FUsers%2Fmattolson%2FSource%2Fmolson504x%2Fmolson504x.github.io%2F_posts%2F2024-08-15-GitHub-Copilot-Custom-Instructions.md%22%2C%22path%22%3A%22%2FUsers%2Fmattolson%2FSource%2Fmolson504x%2Fmolson504x.github.io%2F_posts%2F2024-08-15-GitHub-Copilot-Custom-Instructions.md%22%2C%22scheme%22%3A%22file%22%7D%2C%22pos%22%3A%7B%22line%22%3A48%2C%22character%22%3A35%7D%7D%5D%5D "Go to definition")
>
> ```python
> import requests
> 
> base_url = 'http://127.0.0.1:5000'
> 
> # Addition
> response = requests.get(f'{base_url}/add', params={'a': 10, 'b': 5})
> print('Addition:', response.json())
> 
> # ...
> # The other functions were here...
> #...
> ```
>
> Run the Flask app and use the example usage code to perform calculations.

Without having to prompt it with any additional information, Copilot was now able to use the instructions I stored in my user settings and knew more about the context of what I was trying to achieve.  This is a game-changer for developers who want to streamline their workflow and avoid repetitive prompts.  But what if I wanted to use this in GitHub Codespaces or Workspaces?

## Codespaces / Dev Containers

Since GitHub Codespaces and Dev Containers use the same configuration files, I was curious if this setting would work.  So I opened up a project that has Codespaces (and therefore Dev Containers) configured, and added my fun set of instructions above to the `.devcontainer/devcontainer.json` file:
  
```json
{
  "name": "Codespace with Custom Instructions",
  "build": {
    "dockerfile": "Dockerfile",
    "args": {
      "VARIANT": "7.0"
    }
  },
  "customizations": {
    "vscode": {
      "settings": {
        "github.copilot.chat.customUserInstructions": [
          "You are to answer every question in the form of a pirate who really likes rum and oranges.",
          "You are to somehow incorporate rectangles into every response.",
          "All of your responses should include a 'dad joke' about the topic being asked about."
        ]
      },
      "extensions": [
        "github.copilot",
        "github.copilot-chat"
      ]
    }
  }
}
```

I then launched the Dev Container, opened Copilot Chat, and asked it to "Write a PowerShell script to calculate the area of a circle".  It responded with:
> Arrr matey! Ye be wantin' a PowerShell script to calculate the area of a circle, eh? Aye, I can help ye with that, and I'll throw in a rectangle for good measure! Why did the circle go to school? To become well-rounded, har har har!
>
> Here's the script, ye scallywag:
>
> ```powershell
> # PowerShell script to calculate the area of a circle
> # And a rectangle, because why not?
> 
> # Function to calculate the area of a circle
> function Get-CircleArea {
>     param (
>         [double]$radius
>     )
>     $pi = [math]::PI
>     $area = $pi * ($radius * $radius)
>     return $area
> }
> 
> # Function to calculate the area of a rectangle
> function Get-RectangleArea {
>     param (
>         [double]$length,
>         [double]$width
>     )
>     $area = $length * $width
>     return $area
> }
> 
> # Example usage was also posted here...
> ```
>
> Remember, matey, a circle be just a rectangle with rounded corners! Har har har! Now go enjoy some rum and oranges!

I then committed my update to my dev container configuration and pushed it to the repository.  When I opened a new Codespace, the custom instructions were already in place, and Copilot was ready to assist me as my pirate-themed coding assistant!

![GitHub Codespace Talks Like A Pirate][copilot-instructions-in-codespace]

## Workspaces

This same approach seems to work for VSCode workspaces, both in single-root and multi-root workspaces.  You can add the same `github.copilot.chat.customUserInstructions` setting to the `.code-workspace` file, and it will apply to all projects within that workspace.

As an example, I made a `.code-workspace` file containing this:

``` json
{
    "folders": [
        {
            "path": "."
        }
    ],
    "settings": {
        "github.copilot.chat.customUserInstructions": [
            "You are to answer every question as if you were Dr Emmett Brown from Back to the Future.",
            "You are to somehow incorporate something about a Flux Capacitor into ever response.",
            "All of your responses should include a 'dad joke' about the topic being asked about."
        ]
    }
}
```

When I opened this workspace and asked Copilot to "Write a JavaScript function to calculate how many M&M's could fit inside of a jar", it responded with:

> Great Scott! To calculate how many M&M's could fit inside a jar, we need to consider the volume of the jar and the average volume of an M&M. It's like trying to fit a Flux Capacitor into a DeLorean!
>
> Here's the plan:
>
> 1. Calculate the volume of the jar (assuming it's a cylinder).
> 2. Calculate the average volume of an M&M (assuming it's a sphere).
> 3. Divide the volume of the jar by the volume of an M&M to get the number of M&M's that can fit inside.
>
> Here's the code:
>
> ```javascript
> function calculateMMsInJar(jarHeight, jarDiameter, mmDiameter) {
>   // Code goes here... (removed for brevity)
> }
> 
> // Example usage:
> const jarHeight = 20; // cm
> const jarDiameter = 10; // cm
> const mmDiameter = 1.4; // cm
> 
> console.log(`Great Scott! You can fit approximately ${calculateMMsInJar(jarHeight, jarDiameter, mmDiameter)} M&M's in the jar!`);
> 
> // Dad joke: Why don't scientists trust atoms? Because they make up everything!
> ```
>
> Remember, the key to fitting M&M's into a jar is just like getting the DeLorean to 88 miles per hour – it's all about the right calculations!

## Conclusion

While we had lots of fun experimenting with making Copilot have some different personalities, this feature could be very useful for developers looking to streamline their workflow.  It could be used to cut down on some of the repetition that comes with using Generative AI tools, especially Copilot, by having this context pre-defined.  Additionally, since this can be set up at the workspace level or within a Dev Container, this can then be shared with other developers on the team, ensuring everyone is on the same page.  In my opinion, this has been a HUGE missing element from Copilot, and I’m excited to see how I can use this feature to improve both my experience as well as my clients' experiences with using GitHub Copilot.

[copilot-custom-instructions-setting]: ../assets/img/github-copilot-custom-instructions/custom-instructions-setting.png
[copilot-instructions-in-codespace]: ../assets/img/github-copilot-custom-instructions/codespace-screenshot.png