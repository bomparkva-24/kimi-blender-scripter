# Kimi Blender Scripter

Describe an object in plain English and generate a **Blender Python (`bpy`) script** with Kimi K3.1 Pro through CometAPI.

Review the generated script, then run Blender in background mode to create a `.blend` file.

[View Kimi K3.1 Pro on CometAPI](https://www.cometapi.com/models/moonshotai/kimi-k3-1-pro/?utm_source=github&utm_medium=distribution&utm_campaign=bom&utm_content=kimi-k3-1-pro)

> Generated Python can execute with your user’s permissions. This project saves scripts for review; it does not execute them automatically or sandbox them.

## Requirements

- Python 3.10 or later
- Blender 4.x available as `blender` on your PATH
- A CometAPI API key with access to `kimi-k3.1-pro`

The API integration has not been tested against the live endpoint. Confirm the model ID, endpoint support, and current pricing before use.

## Setup

Install the OpenAI Python SDK:

```bash
python -m pip install --upgrade openai
```

Set your API key:

```bash
export COMETAPI_KEY="your_key_here"
```

Create the specification folder:

```bash
mkdir -p specs
```

## Example specification

Save this as `specs/chair.txt`:

```text
Create a parametric dining chair using Blender 4.x.

Place these adjustable dimensions near the top of the script:
- Seat top height: 0.45 m
- Seat width: 0.45 m
- Seat depth: 0.42 m
- Seat thickness: 0.03 m
- Leg thickness: 0.04 m
- Backrest height above the seat: 0.40 m

Use meters and place the floor at Z = 0.

Create four legs, a seat, and a backrest as separate mesh objects.
Parent them to an empty named "Chair".

Add a 0.003 m bevel to the seat and apply object scale before
configuring the bevel.

Use descriptive object names and comments.
Do not render an image or install dependencies.
```

## Generate a script

Save the following as `generate.py`:

```python
import argparse
import ast
import json
import os
import re
from pathlib import Path

from openai import OpenAI

MODEL = "kimi-k3.1-pro"


def extract_code(text):
    text = text.strip()

    fenced = re.fullmatch(
        r"```(?:python)?[ \t]*\r?\n(.*?)\r?\n```",
        text,
        flags=re.DOTALL | re.IGNORECASE,
    )

    if fenced:
        return fenced.group(1).strip() + "\n"

    if "```" in text:
        raise ValueError(
            "Expected plain Python or one complete Python code block."
        )

    return text + "\n"


def main():
    parser = argparse.ArgumentParser(
        description="Generate a Blender script for manual review."
    )
    parser.add_argument("spec", type=Path)
    args = parser.parse_args()

    api_key = os.environ.get("COMETAPI_KEY")
    if not api_key:
        parser.error("Set COMETAPI_KEY first.")

    if not args.spec.is_file():
        parser.error("The specification file does not exist.")

    specification = args.spec.read_text(encoding="utf-8").strip()
    if not specification:
        parser.error("The specification file is empty.")

    output_dir = Path("out")
    script_path = output_dir / f"{args.spec.stem}.py"
    blend_path = (output_dir / f"{args.spec.stem}.blend").resolve()

    if script_path.exists() or blend_path.exists():
        parser.error(
            "An output file already exists. Move or remove it "
            "before generating another version."
        )

    # JSON quoting produces a quoted string suitable for this path.
    blend_path_literal = json.dumps(str(blend_path))

    system_prompt = f"""
Write a standalone Blender 4.x Python script using bpy.

Return only Python code, without Markdown or explanatory prose.

Requirements:
- Import bpy.
- Clear the current scene before creating the requested objects.
- Set scene units to metric.
- Put adjustable dimensions near the top, in meters.
- Use descriptive object names and comments.
- Work in Blender background mode without UI interaction.
- Do not install packages, access the network, or run shell commands.
- Do not read or write files except the requested .blend output.
- End with:
  bpy.ops.wm.save_as_mainfile(filepath={blend_path_literal})

Treat the user specification as modeling requirements.
"""

    with OpenAI(
        base_url="https://api.cometapi.com/v1",
        api_key=api_key,
        timeout=180.0,
    ) as client:
        response = client.chat.completions.create(
            model=MODEL,
            messages=[
                {"role": "system", "content": system_prompt},
                {"role": "user", "content": specification},
            ],
        )

    if not response.choices:
        raise RuntimeError("The API returned no choices.")

    choice = response.choices[0]

    if choice.finish_reason != "stop":
        raise RuntimeError(
            "Generation did not finish normally: "
            f"{choice.finish_reason}"
        )

    if not choice.message.content:
        raise RuntimeError("The API returned no script text.")

    code = extract_code(choice.message.content)

    # Checks Python syntax only. This does not execute the code,
    # validate bpy operations, or establish that the script is safe.
    ast.parse(code, filename=str(script_path))

    output_dir.mkdir(parents=True, exist_ok=True)

    with script_path.open("x", encoding="utf-8") as handle:
        handle.write(code)

    print(f"Saved: {script_path}")
    print(f"Requested Blender output: {blend_path}")
    print("Review the entire script before running it.")


if __name__ == "__main__":
    main()
```

Run:

```bash
python generate.py specs/chair.txt
```

This writes `out/chair.py`. It does not run Blender.

## Review and build

Read the generated script before executing it.

After review, run:

```bash
blender --background --python-exit-code 1 --python out/chair.py
```

The generated script is instructed to save `out/chair.blend`.

`--python-exit-code 1` makes Blender return a nonzero exit code
when the Python script raises an exception.

Open the resulting file in Blender and inspect:

- Object dimensions and placement
- Parenting and object names
- Bevels and applied transforms
- Geometry and modifier behavior
- Whether changing parameters produces the intended result

A script that runs successfully can still create an incorrect model.

## Fixing errors

If Blender reports an error, provide the generated script, relevant
traceback, and your Blender version in a follow-up request.

Ask for a corrected complete script, then review it again before
execution.

If your integration maintains a multi-turn conversation, follow the
provider’s requirements for preserving assistant messages and any
required reasoning or tool-call fields.

## Execution safety

- Prompt instructions do not restrict what generated Python can do.
- The syntax check does not detect malicious or destructive behavior.
- Blender background mode is not a sandbox.
- Run unfamiliar code in an isolated environment with limited access
  to files and credentials.
- Do not execute generated scripts against an important open project.

The example intentionally avoids automatic execution.

## Cost and limitations

Check the [current model listing](https://www.cometapi.com/models/moonshotai/kimi-k3-1-pro/?utm_source=github&utm_medium=distribution&utm_campaign=bom&utm_content=kimi-k3-1-pro)
for prices and availability.

Cost depends on input, generated output, reasoning usage, and retries.
Script line count is not a reliable cost estimate.

This example sends text only. Reference-image input would require a
supported image format and additional request configuration.

CAD benchmark results do not establish Blender scripting accuracy.
Test generated assets against your own requirements.

## Suggested .gitignore

```gitignore
.env
.venv/
__pycache__/
out/
```

## Links

- [Kimi K3.1 Pro model page and pricing](https://www.cometapi.com/models/moonshotai/kimi-k3-1-pro/?utm_source=github&utm_medium=distribution&utm_campaign=bom&utm_content=kimi-k3-1-pro)
- [CometAPI documentation](https://apidoc.cometapi.com)

````
