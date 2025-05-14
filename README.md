
import re, string, sys

class BananaInterpreter:
    def __init__(self):
        self.environment = {}
        self.functions = {}
        self.use = {}
        self._in_function_definition = False
        self._current_func_name = None
        self._current_func_body = []
        self._lines = []
    
    def interpret(self, code):
        self._lines = code.strip().split('\n')
        i = 0
        
        while i < len(self._lines):
            line = self._lines[i].strip()
            
            if not line:
                i += 1
                continue

            if self._in_function_definition:
                if line == "}":
                    self.functions[self._current_func_name] = {"name": self._current_func_name, "body": "\n".join(self._current_func_body)}
                    self._in_function_definition = False
                    self._current_func_name = None
                    self._current_func_body = []
                else:
                    self._current_func_body.append(line)
                i += 1
            elif line.startswith("FUNC"):
                match = re.match(r"FUNC\s+(\w+)\s*\(\)\s*\{", line)
                if match:
                    self._current_func_name = match.group(1)
                    self._in_function_definition = True
                    self._current_func_body = []
                    i += 1
                else:
                    raise SyntaxError(f"Funzione definita in modo errato: {line}")
                    i += 1
            elif line.startswith("LET"):
                self.process_variable_declaration(line)
                i += 1
            elif line.startswith("PRINT"):
                self.process_print(line)
                i += 1
            elif re.match(r"^\w+\(\)$", line):
                self.process_function_call(line, i)
                i += 1
            elif line.startswith("IF"):
                i = self.process_if(line, i)    
                i += 1
            elif line.startswith("USE"):
                self.process_import(line)
                i += 1    
            elif line.startswith("SKIP"):
                match  = re.match(r"SKIP\s*(\d+)", line)
                if match:
                    value = match.group(1)
                    i += int(value)
                else:
                    raise SyntaxError(f"Sintassi SKIP sbagliata: {line}")    
            elif line.startswith("DO"):
                self.doing(line)
                i += 1        
            else:
                raise SyntaxError(f"Errore Comando\nComando Banana non riconosciuto: {(line)}")
                i += 1
                
    def execute_line(self, line, i):
        if line.startswith("LET"):
            self.process_variable_declaration(line)
        elif line.startswith("PRINT"):
            self.process_print(line)
        elif line.startswith("FUNC"):
            self.process_function_definition(line)
        elif re.match(r"^\w+\(\)$", line): # Riconosce una chiamata di funzione senza argomenti
            self.process_function_call(line, i)
        elif line.startswith("IF"):
            i = self.process_if(line, i)
        elif line.startswith("USE"):
            self.process_import(line)  
        elif line.startswith("SKIP"):
            match  = re.match(r"SKIP\s*(\d+)", line)
            if match:
                value = match.group(1)
                i += int(value)
            else:
                raise SyntaxError(f"Sintassi SKIP sbagliata")
        elif line.startswith("DO"):
            self.doing(line)
        else:
            raise SyntaxError(f"Comando Banana non riconosciuto: {line}")
            
    def process_variable_declaration(self, line):
        match = re.match(r"LET\s+(\w+):\s+(\w+)\s*=\s*(\S+)", line)
        if match:
            variable = match.group(1)
            declared_type = match.group(2)
            value_str = match.group(3)
            
            if declared_type == "input":
                try:
                    value = input(f"{variable}: ")
                    if value_str not in ["int", "str", "float", "null"]:
                        raise TypeError(f"Il tipo di input non è valido: {line}")
                        sys.exit()
                    
                    if value_str == "int":
                        calculate = int(value)    
                    elif value_str == "float":
                        calculate = float(value)
                    elif value_str == "null":
                        value = None
                    else:
                        calculate = str(value)    
                    self.environment[variable] = {"type": declared_type, "value": value}
                except ValueError:
                    raise InputError(f"Input non valido: {line}")
            elif declared_type == "int":
                try:
                    value = int(value_str)
                    self.environment[variable] = {"type": declared_type, "value": value}
                except TypeError:
                    raise TypeError(f"Errore: '{value_str}' non è un intero valido per la variabile '{variable}'")
            elif declared_type == "str":
                try:
                    match_str = re.match(r'\s*"([^"]*)"\s*', value_str)
                    if match_str.group(1):
                        value = str(match_str.group(1))
                        self.environment[variable] = {"type": declared_type, "value": value}
                    else:
                        raise SyntaxError(f"Variabile\nVariabile definita in modo errato: {line}")
                except Exception:
                    raise Exception(f"Errore imprevisto durante la dichiarazione e assegnazione del valore ad una variabile: {line}")
            elif declared_type == "ref":
                try:
                    if value_str in self.environment:
                        self.environment[variable] = {
                            "type": self.environment[value_str]['type'],
                            "value": self.environment[value_str]['value']
                        }    
                    else:
                        raise NameError(f"Errore tipo dichiarato\nTipo dichiarato contenente valore nuovo: {line}")    
                except Exception:
                    raise NameError(f"La variabile non è stata definita correttamente: {line}")
            elif declared_type == "float":
                try:
                    value = float(value_str)
                    self.environment[variable] = {"type": declared_type, "value": value}
                except ValueError:
                    raise TypeError(f"Il valore '{value_str}' non è valido per la variabile '{variable}'. Forse intendevi dichiarare la variabile come: {type(value_str).__name__}")           
            elif declared_type == "math":
                try:
                    expr = value_str
                    for declared_type in self.environment:
                        expr = expr.replace(declared_type, str(self.environment[declared_type]['value']))
        
                    value = eval(expr)
                    self.environment[variable] = {"type": declared_type, "value": value}
                except:
                    raise ArithmeticError(f"Errore nel calcolo: {value_str}")
            elif declared_type == "list":
                try:
                    value = list(value_str.split(","))
                    self.environment[variable] = {"type": declared_type, "value": value}
                except (TypeError, Exception):
                    raise TypeError(f"Impossibile trasformare in lista la variabile: {line}") 
            elif declared_type == "auto":
                if value_str.startswith('"') and value_str.endswith('"'):
                    self.environment[variable] = {"type": "int", "value": len(value_str.strip('"'))}
                else:
                    self.environment[variable] = {"type": declared_type, "value": value_str}
                return        
            elif declared_type == "null":
                self.environment[variable] = {"type": declared_type, "value": value_str}    
        else:
            raise SyntaxError(f"Errore di sintassi nella dichiarazione LET: {line}")
    
    def process_print(self, line):
        match = re.match(r"PRINT\s+(\w+)", line)
        if match:
            variable = match.group(1)
            if variable in self.environment:
                print(self.environment[variable]["value"])
            else:
               print(variable)
        else:
            raise SyntaxError(f"Errore di sintassi in PRINT: {line}")
            
    def process_function_definition(self, line):
        match = re.match(r"FUNC\s+(\w+)\s*\(\)\s*\{(.*)\}", line, re.DOTALL)
        if match:
            func_name = match.group(1)
            func_body = match.group(2).strip()
            self.functions[func_name] = {"name": func_name, "body": func_body}
        else:
            raise SyntaxError(f"Funzione definita in modo errato: {line}")   
    
    def process_function_call(self, line, i):
        match = re.match(r"(\w+)\(\)", line)
        if match:
            call = match.group(1)
            if call in self.functions:
                func_body = self.functions[call]['body'] 
                
                if isinstance(func_body, str):
                    body_lines = func_body.split('\n')
                else:
                    body_lines = func_body    
                
                for body_line in body_lines:
                    body_line = body_line.strip()
                    if body_line:
                        self.execute_line(body_line, i)
                    
            else:
                raise NameError(f"Impossibile call\nNessuna funzione definita col nome: {line}")  
                
        else:
            raise SyntaxError(f"Error call\nChiamata eseguita in modo errato: {line}")
            
    def process_if(self, line, current_idx):
        """Nuova versione con supporto per blocchi indentati"""
        match = re.match(r"IF\s+(.+?)\s*(==|!=|>=|<=|>|<)\s*(.+?)(?:\s*\{)?", line)
        if not match:
            return current_idx + 1
    
        left, op, right = match.groups()
        left_val = self.get_value(left.strip())
        right_val = self.get_value(right.strip())
    
        op = op.replace("=", "==").replace("=>", ">=").replace("=<", "<=")
        
        try:
            condition_met = eval(
                f'"{left_val}" {op} "{right_val}"' if isinstance(left_val, str) or isinstance(right_val, str)
                else f"{left_val} {op} {right_val}"
            )
        except:
            return current_idx + 1
    
        if condition_met:
        # Esegui tutte le righe indentate o fino a }/END
            next_idx = current_idx + 1
            while next_idx < len(self._lines):
                next_line = self._lines[next_idx].strip()
            
            # Condizioni di uscita dal blocco
                if next_line in ("}", "END") or not self._is_indented(next_idx):
                    break
                
                if next_line:  # Ignora righe vuote
                    self.execute_line(next_line, current_idx)
                next_idx += 1
            return next_idx + 1 if next_idx < len(self._lines) else next_idx
        else:
        # Salta tutto il blocco
            return self._find_block_end(current_idx)
    
    def get_value(self, expr):
        if expr in self.environment:
            return self.environment[expr]['value']
        try:
            return float(expr) if '.' in expr else int(expr)  # <-- Questa riga
        except:
            return expr
    
    # Conversione numerica
        try:
            return int(expr)
        except ValueError:
            try:
                return float(expr)
            except ValueError:
                return expr
    
    def _is_indented(self, line_idx):
        """Controlla se la riga è indentata rispetto all'IF"""
        if line_idx >= len(self._lines):
            return False
        return self._lines[line_idx].startswith(("    ", "\t"))
    
    def _find_block_end(self, start_idx):
        """Trova la fine del blocco (} o END)"""
        for i in range(start_idx + 1, len(self._lines)):
            if self._lines[i].strip() in ("}", "END"):
                return i + 1
        return len(self._lines)  # Restituisce come stringa  
    
    def process_import(self, line):
        match = re.match(r"USE\s+(\w+)", line)  # \s+ richiede almeno uno spazio
        if not match:
            raise SyntaxError(f"Errore di sintassi in USE: {line}")

        module_name = match.group(1)
        try:
        # Importa il modulo dinamicamente
            imported = __import__(module_name)
            self.use[module_name] = imported
        except ImportError as e:
            raise ImportError(f"Errore durante l'import di {module_name}: {str(e)}")
        except ModuleNotFoundError as m:
            raise ModuleNotFoundError(f"Modulo non trovato di nome {module_name}: {str(m)}")    
    
    def doing(self, line):
        match = re.match(r"DO\s+(\w+)", line)
        if match:
            if match.group(1) in self.environment:
                name = match.group(1)
                value = eval("self.environment[name]['value']")
                self.environment[name] = {
                    "type": self.environment[name]['type'],
                    "value": eval(value)
                }
            else:
                raise KeyError(f"{match.group(1)} non esiste come variabile: {line}")    
        else:
            raise SyntaxError(f"Sintassi DO errata: {line}")        
                        
# Esempio di utilizzo Banana
banana_code = """
USE random
LET x: null = __import__("random")
DO x
LET a: null = random.randint(0,6)
DO a
PRINT a
"""
interpreter = BananaInterpreter()
interpreter.interpret(banana_code)
