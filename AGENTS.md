
# Jörg Jooss Blog

## Development Environment
This is the source code for a blog built with Hugo and the Blowfish theme. No build-specific activity is required or expected. All source code consists of Markdown files or website content such as images, CSS, or JavaScript. The blog uses leaf bundles for individual blog posts. All other Markdown files like `about.md` are static pages.

## Agent Objectives and Commands

- Your task is to spellcheck, correct grammar, and improve the text for conciseness, accuracy, and clarity only. 

- All text is in American English.

- Apply changes only to either a file provided in the prompt, or, if no file is provided, use the currently opened `index.md` file in the editor. If the current file is not called `index.md`, stop immediately.

- If you receive the prompt `review`, fix spelling and correct grammar in text.

- If you receive the prompt `copyedit`, improve the text regarding technical accuracy, clarity, readability, and consistency. You may read other local `index.md` files the current file refers to.  

- If you receive the prompt `summary`, add a concise summary at the end of the file under the heading `## Summary` in past tense. If there is already a summary, `copyedit` the summary only.

- There are no build or test steps you need to perform. 

## Detailed Edit Instructions
- Ignore all CSS, JavaScript, or other stand-alone files you consider source code.
- Only edit Markdown files called `index.md` under the `content/posts` directory.
- Do not change any source code in Markdown code blocks like ```go...```. Report any issues you find in code blocks only as a direct response to the user.
- Do not change the Hugo front matter of a blog article, i.e., all text at the beginning of a file between `+++...+++`.
- Do not change any versions mentioned in the text, for example versions of programming languages (Go 1.25), LLMs (GPT-5-Mini), or applications (Visual Studio 2026), even if you are not aware these versions exist.
