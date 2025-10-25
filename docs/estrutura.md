# ZSec — Estrutura do projeto e painel CLI

Este documento contém um scaffold prático para o seu projeto **ZSec** (nome provisório). Inclui: árvore de diretórios proposta, scripts de instalação, um painel CLI em Python (menu) para carregar módulos, template de módulo, e instruções rápidas para rodar no Termux.

---

## 1) Estrutura sugerida

```
ZSec/
├── README.md
├── install.sh            # instala dependências no Termux
├── zsec.sh               # launcher em bash (opcional)
├── requirements.txt
├── core/
│   ├── __init__.py
│   ├── main.py           # painel CLI principal
│   ├── loader.py         # carregador dinâmico de módulos
│   └── utils.py          # helpers (log, format, network helpers)
├── modules/              # módulos (cada um é um pacote)
│   ├── recon/
│   │   ├── __init__.py
│   │   └── recon.py      # scanner / whois / banner
│   ├── hashes/
│   │   ├── __init__.py
│   │   └── hashcheck.py
│   └── websec/
│       ├── __init__.py
│       └── xss_check.py
└── data/
    └── logs/             # logs e relatórios gerados
```

Use essa organização para que cada módulo tenha sua própria responsabilidade e possa ser carregado dinamicamente pelo `core.loader`.

---

## 2) `install.sh` (Termux)

```bash
#!/usr/bin/env bash
set -e
pkg update -y && pkg upgrade -y
pkg install -y python git nmap curl wget openssh netcat
pip install --upgrade pip
pip install -r requirements.txt || true
echo "Instalação concluída. Rode: python core/main.py"
```

Crie `requirements.txt` com (exemplo):

```
requests
rich
python-dotenv
```

Adicione `scapy` se for usar manipulação avançada de pacotes (pode precisar de privilégios).

---

## 3) `core/main.py` — Painel CLI (menu) básico em Python

```python
#!/usr/bin/env python3
import os
import sys
from importlib import import_module
from core import loader
from rich.console import Console
from rich.table import Table

console = Console()

def show_menu(modules):
    table = Table(title="ZSec - Painel de Módulos")
    table.add_column("#", justify="right")
    table.add_column("Módulo")
    table.add_column("Descrição")
    for i, m in enumerate(modules, start=1):
        table.add_row(str(i), m["name"], m.get("desc", "-"))
    table.add_row("q", "Sair", "Fechar ZSec")
    console.print(table)


def main():
    modules = loader.discover_modules("modules")
    while True:
        os.system("clear")
        show_menu(modules)
        choice = input("Escolha um módulo (número) > ").strip()
        if choice.lower() in ("q", "exit", "sair"):
            print("Saindo...")
            sys.exit(0)
        try:
            idx = int(choice) - 1
            if idx < 0 or idx >= len(modules):
                raise ValueError
        except ValueError:
            print("Escolha inválida.")
            input("Enter para continuar...")
            continue
        mod = modules[idx]
        # cada módulo expõe run() como entrypoint
        module_obj = import_module(mod["import_path"])
        try:
            module_obj.run()
        except Exception as e:
            console.print(f"Erro no módulo: {e}")
        input("Enter para voltar ao menu...")

if __name__ == '__main__':
    main()
```

Esse painel usa `core.loader.discover_modules` para obter módulos; veja o exemplo abaixo.

---

## 4) `core/loader.py` — descoberta simples de módulos

```python
import os
import importlib


def discover_modules(base_dir='modules'):
    modules = []
    for name in os.listdir(base_dir):
        path = os.path.join(base_dir, name)
        if os.path.isdir(path) and os.path.exists(os.path.join(path, '__init__.py')):
            # convenção: cada __init__.py define meta = {name, desc, entry}
            import_path = f"{base_dir.replace('/', '.')}.{name}.{name}"
            try:
                mod = importlib.import_module(import_path)
                meta = getattr(mod, 'META', None)
                modules.append({
                    'name': meta.get('name', name) if meta else name,
                    'desc': meta.get('desc', '' ) if meta else '',
                    'import_path': import_path
                })
            except Exception as e:
                print(f"Falha ao carregar módulo {name}: {e}")
    return modules
```

Adapte `import_path` conforme a sua convenção de nomes.

---

## 5) Template de módulo (`modules/recon/recon.py`)

```python
# modules/recon/recon.py
"""Módulo: Recon
Exemplo simples de scanner com nmap ou socket."""

META = {
    'name': 'Recon - Scan simples',
    'desc': 'Faz ping sweep e banner grabbing',
}

import subprocess
from rich.console import Console
console = Console()


def run():
    console.print('[bold]Recon module[/bold]')
    target = input('Alvo (ex: 192.168.1.0/24 or 192.168.1.10) > ').strip()
    if not target:
        print('Nenhum alvo informado.')
        return
    # exemplo simples: nmap (requer nmap instalado)
    try:
        subprocess.run(['nmap', '-sV', target])
    except FileNotFoundError:
        console.print('nmap não encontrado. Instale usando: pkg install nmap')
```

Cada módulo expõe `META` e `run()` — o painel chama `run()`.

---

## 6) `zsec.sh` — launcher em bash

```bash
#!/usr/bin/env bash
PYTHON=python
DIR=$(cd "$(dirname "$0")" && pwd)
$PYTHON "$DIR/core/main.py"
```

Torna mais rápido rodar com `./zsec.sh` (dê `chmod +x zsec.sh`).

---

## 7) Boas práticas e ideias UX

* Tenha módulos pequenos e especializados (recon, websec, crypto, utils).
* Use `rich` para tabelas e outputs bonitos no terminal.
* Registre relatórios em `data/logs/` automáticamente.
* Crie um `--headless` mode para rodar scripts via cron/CI.
* Controle de permissões: documente claramente quando um módulo precisa de root ou privilégios especiais.

---

## 8) Próximos passos que posso ajudar agora

* Gerar o código completo do `core/main.py` + `core/loader.py` + 2 módulos de exemplo prontos para executar no Termux.
* Escrever um `Makefile` simples ou script para empacotar módulos.
* Adicionar autenticação simples / histórico de comandos.

Me diz qual desses você quer que eu gere **agora** que eu já crio os arquivos de exemplo prontos pra você copiar/colar.

