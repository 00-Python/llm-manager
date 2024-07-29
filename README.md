### Project Structure

```
llm-cli/
├── llm_cli.py
├── models/
│   ├── __init__.py
│   └── llm_manager.py
├── services/
│   ├── __init__.py
│   ├── base_assistant.py
│   ├── openai_assistant.py
│   ├── claude_assistant.py
│   └── copilot_assistant.py
├── commands/
│   ├── __init__.py
│   ├── load_code.py
│   ├── debug.py
│   ├── document.py
│   ├── chat.py
│   ├── code_review.py
│   ├── assistant_factory.py
├── config/
│   └── config.json
├── requirements.txt
└── README.md
```

### Step 1: Update the `LLMManager` to Use SQLite3

#### `models/llm_manager.py`

```python
import sqlite3
from typing import Optional

DB_PATH = 'config/accounts.db'

class LLMManager:
    def __init__(self):
        self.conn = sqlite3.connect(DB_PATH)
        self.create_tables()

    def create_tables(self):
        with self.conn:
            self.conn.execute('''
                CREATE TABLE IF NOT EXISTS accounts (
                    id INTEGER PRIMARY KEY,
                    name TEXT UNIQUE NOT NULL,
                    api_key TEXT NOT NULL,
                    provider TEXT NOT NULL
                )
            ''')
            self.conn.execute('''
                CREATE TABLE IF NOT EXISTS active_account (
                    id INTEGER PRIMARY KEY,
                    account_name TEXT,
                    provider TEXT
                )
            ''')

    def add_account(self, name: str, api_key: str, provider: str):
        with self.conn:
            self.conn.execute('''
                INSERT INTO accounts (name, api_key, provider) VALUES (?, ?, ?)
            ''', (name, api_key, provider))

    def switch_account(self, name: str, provider: str):
        with self.conn:
            self.conn.execute('DELETE FROM active_account')
            self.conn.execute('''
                INSERT INTO active_account (account_name, provider) VALUES (?, ?)
            ''', (name, provider))

    def get_current_account(self) -> Optional[dict]:
        cursor = self.conn.execute('SELECT account_name, provider FROM active_account LIMIT 1')
        row = cursor.fetchone()
        if row:
            return {'name': row[0], 'provider': row[1]}
        return None

    def get_api_key(self, name: str) -> Optional[str]:
        cursor = self.conn.execute('SELECT api_key FROM accounts WHERE name = ?', (name,))
        row = cursor.fetchone()
        if row:
            return row[0]
        return None

    def close(self):
        self.conn.close()
```

### Step 2: Update the Commands to Include Provider

#### `llm_cli.py`

```python
import cmd
import subprocess
from models.llm_manager import LLMManager
from commands.load_code import LoadCodeCommand
from commands.debug import DebugCommand
from commands.document import DocumentCommand
from commands.chat import ChatCommand
from commands.code_review import CodeReviewCommand

class LLMCLI(cmd.Cmd):
    intro = "Welcome to the LLM CLI. Type help or ? to list commands.\n"
    prompt = "(llm-cli) "

    def __init__(self):
        super().__init__()
        self.manager = LLMManager()
        self.load_code_cmd = LoadCodeCommand(self.manager)
        self.debug_cmd = DebugCommand(self.manager)
        self.document_cmd = DocumentCommand(self.manager)
        self.chat_cmd = ChatCommand(self.manager)
        self.code_review_cmd = CodeReviewCommand(self.manager)

    def do_add_account(self, arg):
        'Add an account: add_account name api_key provider'
        try:
            name, api_key, provider = arg.split()
            self.manager.add_account(name, api_key, provider)
            print(f"Account {name} added successfully for provider {provider}.")
        except ValueError:
            print("Usage: add_account name api_key provider")

    def do_switch_account(self, arg):
        'Switch to an account: switch_account name provider'
        try:
            name, provider = arg.split()
            self.manager.switch_account(name, provider)
            print(f"Switched to account {name} for provider {provider}.")
        except ValueError as e:
            print("Usage: switch_account name provider")

    def do_load_code(self, arg):
        'Load a code file: load_code file_path'
        self.load_code_cmd.execute(arg)

    def do_debug(self, arg):
        'Debug code: debug code'
        self.debug_cmd.execute(arg)

    def do_document(self, arg):
        'Generate documentation: document code'
        self.document_cmd.execute(arg)

    def do_chat(self, arg):
        'Chat with assistant: chat message'
        self.chat_cmd.execute(arg)

    def do_code_review(self, arg):
        'Review code: code_review code'
        self.code_review_cmd.execute(arg)

    def do_quit(self, arg):
        'Quit the CLI: quit'
        self.manager.close()
        print("Quitting...")
        return True

    def default(self, line):
        'Handle unrecognized commands as system commands'
        try:
            subprocess.run(line, shell=True, check=True)
        except subprocess.CalledProcessError as e:
            print(f"Command '{line}' failed with return code {e.returncode}")
        except Exception as e:
            print(f"Error executing command '{line}': {e}")

if __name__ == '__main__':
    LLMCLI().cmdloop()
```

### Step 3: Update the Assistant Factory

#### `commands/assistant_factory.py`

```python
from services.openai_assistant import OpenAIAssistant
from services.claude_assistant import ClaudeAssistant
from services.copilot_assistant import CopilotAssistant
from models.llm_manager import LLMManager

class AssistantFactory:
    def __init__(self, manager: LLMManager):
        self.manager = manager

    def create_assistant(self):
        current_account = self.manager.get_current_account()
        if not current_account:
            raise ValueError("No account is currently active. Please switch to an account first.")

        api_key = self.manager.get_api_key(current_account['name'])
        provider = current_account['provider']

        if provider == "openai":
            return OpenAIAssistant(api_key)
        elif provider == "claude":
            return ClaudeAssistant(api_key)
        elif provider == "copilot":
            return CopilotAssistant(api_key)
        else:
            raise ValueError("Unsupported LLM account type.")
```

### Commands Implementations

These implementations remain the same as previously described but use the `LLMManager` to get the assistant:

#### `commands/load_code.py`

```python
class LoadCodeCommand:
    def __init__(self, manager):
        self.manager = manager

    def execute(self, file_path):
        try:
            with open(file_path, 'r') as file:
                code = file.read()
            print(f"Code loaded from {file_path}:\n{code}")
            self.manager.current_code = code
        except Exception as e:
            print(f"Error loading file: {e}")
```

#### `commands/debug.py`

```python
class DebugCommand:
    def __init__(self, manager):
        self.manager = manager

    def execute(self, code):
        assistant = AssistantFactory(self.manager).create_assistant()
        prompt = f"Debug the following code:\n{code}"
        response = assistant.assist(prompt)
        print(f"Debugging suggestions:\n{response}")
```

#### `commands/document.py`

```python
class DocumentCommand:
    def __init__(self, manager):
        self.manager = manager

    def execute(self, code):
        assistant = AssistantFactory(self.manager).create_assistant()
        prompt = f"Generate documentation for the following code:\n{code}"
        response = assistant.assist(prompt)
        print(f"Documentation:\n{response}")
```

#### `commands/chat.py`

```python
class ChatCommand:
    def __init__(self, manager):
        self.manager = manager

    def execute(self, message):
        assistant = AssistantFactory(self.manager).create_assistant()
        prompt = f"Chat with the assistant:\n{message}"
        response = assistant.assist(prompt)
        print(f"Chat response:\n{response}")
```

#### `commands/code_review.py`

```python
class CodeReviewCommand:
    def __init__(self, manager):
        self.manager = manager

    def execute(self, code):
        assistant = AssistantFactory(self.manager).create_assistant()
        prompt = f"Review the following code:\n{code}"
        response = assistant.assist(prompt)
        print(f"Code review suggestions:\n{response}")
```

### Step 5: Requirements and Installation

1. **Install dependencies:**
   ```bash
   pip install -r requirements.txt
   ```

2. **Run the interactive CLI tool:**
   ```bash
   python llm_cli.py
   ```

### Step 6: Usage Examples

1. **Add an account:**
   ```sh
   (llm-cli) add_account openai_account YOUR_OPENAI_API_KEY_HERE openai
   ```

2. **Switch to an account:**
   ```sh
   (llm-cli) switch

_account openai_account openai
   ```

3. **Load a code file:**
   ```sh
   (llm-cli) load_code path/to/your/code.py
   ```

4. **Debug code:**
   ```sh
   (llm-cli) debug "def my_function(x): return x * 2"
   ```

5. **Generate documentation:**
   ```sh
   (llm-cli) document "def my_function(x): return x * 2"
   ```

6. **Chat with the assistant:**
   ```sh
   (llm-cli) chat "Hello, how can you help me?"
   ```

7. **Review code:**
   ```sh
   (llm-cli) code_review "def my_function(x): return x * 2"
   ```

8. **Run system commands:**
   ```sh
   (llm-cli) ls -la
   ```

9. **Quit the CLI:**
   ```sh
   (llm-cli) quit
   ```

This setup now includes account management with SQLite3 and ensures that the tool can handle both custom and system commands, providing a comprehensive solution for professional software development workflows.
