# Power Analysis Warmup — vazamento do bit menos significativo da S-box

Este script quebra uma chave de 16 bytes (AES-like) explorando um servidor
que, para cada byte de um plaintext cifrado, **vaza se o resultado do S-box
foi par ou ímpar**. É a versão "de aquecimento" do ataque de canal lateral,
bem mais simples que o CPA (não precisa de estatística sobre centenas de
amostras — cada byte pode ser determinado quase com certeza).

---

## 1. A ideia por trás do ataque

### O que o servidor faz

O código-fonte do desafio (`encrypt.py`) mostrado pelo organizador revela
exatamente o que acontece:

```python
def leaky_aes_secret(data_byte, key_byte):
    out = Sbox[data_byte ^ key_byte]
    leak_buf.append(out & 0x01)   # guarda so o ultimo bit do resultado
    return out

def encrypt_and_leak(plaintext):
    ciphertext = encrypt(plaintext, SECRET_KEY)
    return leak_buf.count(1)      # soma dos bits guardados, 1 numero so
```

Ou seja: para os 16 bytes do plaintext, o servidor calcula
`Sbox[plaintext[i] XOR chave[i]]`, guarda **só o bit menos significativo**
desse resultado (0 se for par, 1 se for ímpar), e no final devolve **a soma
desses 16 bits** — um único número entre 0 e 16.

### Por que isso é suficiente para quebrar a chave

O número total sozinho não diria muito, mas o ataque usa uma técnica de
**isolamento por diferença**:

1. Primeiro, manda um plaintext todo zero e guarda o total (`baseline`).
2. Para descobrir o byte `i` da chave, manda vários plaintexts onde **só o
   byte `i` muda** (os outros continuam zero). Qualquer variação no total
   em relação ao baseline só pode vir daquele byte, porque os outros 15
   bytes do S-box não mudaram.
3. Com algumas amostras (10-15 valores diferentes de `x` testados naquele
   byte), dá para comparar o padrão observado com o padrão que **cada uma
   das 256 chaves possíveis prediria**, e sobra só uma chave consistente
   com tudo que foi medido.

Isso é bem mais "limpo" que o CPA: aqui não tem ruído físico de verdade, o
vazamento é determinístico (sempre o mesmo valor para a mesma entrada), só
que agregado em uma soma. Por isso bastam algumas dezenas de consultas por
byte, em vez de centenas de traces.

---

## 2. Estrutura do código

### Constantes

```python
HOST = "chatelaine.cylabacademy.net"
PORT = 20335
```
Endereço do servidor do desafio.

```python
Sbox = (0x63, 0x7C, ...)
```
A mesma S-box do AES, copiada diretamente do código-fonte fornecido pelo
desafio — é pública, então usá-la aqui não é "trapaça", é só reaproveitar a
mesma tabela que o servidor usa internamente.

---

### Comunicação com o servidor

#### `query(hex_pt)`

Faz **uma consulta**: abre uma conexão nova, lê o prompt do servidor
("Please provide 16 bytes... :"), manda o plaintext em hexadecimal, e lê a
resposta até encontrar o texto `"leakage result:"`, extraindo o número que
vem depois.

Como o servidor fecha a conexão após responder (confirmado pela
transcrição do desafio), cada consulta precisa abrir uma conexão TCP nova
— por isso essa função usa `socket.create_connection` dentro de um `with`,
que garante que a conexão é fechada corretamente ao final.

#### `plaintext_hex(byte_index, value)`

Função auxiliar: gera uma string hex de 16 bytes, todos zero, exceto no
índice `byte_index`, onde coloca `value`. Por exemplo,
`plaintext_hex(3, 0xAB)` gera um plaintext com zero em todo lugar, menos no
quarto byte, que vale `0xAB`.

Essa é a peça central do ataque: isolar a variação de um único byte por vez.

---

### O ataque, byte por byte

#### `main()`

1. **Baseline**: consulta o servidor com plaintext todo zero e guarda o
   total (`L0`). Esse valor serve de referência — qualquer mudança em
   relação a ele, ao variar um byte específico, é causada só por aquele
   byte.

2. **Para cada byte `i` de 0 a 15**:
   - Testa um conjunto inicial de valores (`samples = range(1, 12)`),
     cada um isolado com `plaintext_hex(i, x)`.
   - Para cada valor `x`, guarda `measurements[x] = m(x) - L0` — a
     diferença em relação ao baseline.
   - **Testa as 256 chaves possíveis** (`k` de 0 a 255): para cada uma,
     calcula o que a teoria prevê (`bitx - bit0`, usando a S-box) e compara
     com o que foi medido de verdade. Só sobram como "candidatas" as chaves
     cuja previsão bate com **todas** as medições feitas.
   - Se sobrar mais de uma candidata (raro, mas pode acontecer com poucas
     amostras), o script testa valores extras de `x` (`range(12, 40)`) até
     que só reste uma.

3. Ao final, junta os 16 bytes encontrados numa string hex e monta a flag
   no formato `academy{<chave>}`.

---

## 3. Por que esse design específico

| Decisão de design | Motivo |
|---|---|
| Isolar um byte por vez (resto do plaintext em zero) | Permite atribuir qualquer mudança no total exclusivamente àquele byte, sem precisar separar a contribuição dos outros 15 |
| Comparar contra baseline, não valor absoluto | O vazamento é uma soma agregada; só a *diferença* em relação a uma referência conhecida isola a informação de um byte |
| Testar poucos valores de `x` primeiro, expandir só se precisar | A S-box do AES tem bom comportamento de difusão — poucas amostras já eliminam quase todas as 256 hipóteses erradas, então não vale a pena gastar 256 consultas por byte de cara |
| Reconexão por consulta | O servidor fecha a conexão a cada resposta, então não dá para manter uma sessão longa aberta |

---

## 4. Limitações e cuidados

- **Falta de robustez a ruído**: diferente do CPA, este ataque assume que o
  vazamento é **determinístico** (a mesma entrada sempre dá o mesmo
  resultado). Se o servidor tivesse algum componente aleatório no
  vazamento, o método de "só sobra uma candidata" quebraria facilmente.
- **Número de consultas por byte**: no pior caso (quando a primeira
  rodada de amostras não é suficiente), o script pode gastar bem mais
  consultas por byte. Isso ainda é rápido, mas quanto maior a instabilidade
  da rede, mais chances de atrasos.
- **Dependência do código-fonte do desafio**: este ataque só funciona
  porque tivemos acesso ao `encrypt.py` e sabíamos exatamente a fórmula do
  vazamento (`Sbox[pt^k] & 1`). Em um cenário de side-channel real, a
  primeira etapa costuma ser justamente descobrir essa relação.
