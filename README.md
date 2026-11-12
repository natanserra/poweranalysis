# Mini RSA — ataque de raiz e-ésima (expoente pequeno)

Este script quebra uma cifra RSA onde o expoente público `e` é **pequeno**
(tipicamente 3) e a mensagem não tem padding suficiente para evitar o
"estouro" controlado de `M^e` em relação a `N`.

---

## 1. A ideia por trás do ataque

No RSA normal, `c = M^e mod N`. Quando `e` é pequeno e `M` não é
"inflado" o suficiente por um bom esquema de padding, pode acontecer de
`M^e` ser **apenas um pouco maior** que `N` — ou seja, a operação de
módulo só subtraiu `N` umas poucas vezes.

Formalmente:
```
c = M^e mod N = M^e - k*N,  para algum inteiro k pequeno (0, 1, 2, ...)
```

Reorganizando:
```
M^e = c + k*N
```

Se testarmos `k = 0, 1, 2, ...` e, para cada um, verificarmos se
`c + k*N` é uma **potência e-ésima perfeita**, eventualmente encontramos o
`k` certo — e a raiz e-ésima exata desse número é a mensagem `M`.

Esse é um ataque bem mais rápido que fatorar `N`: não precisa descobrir
`p` e `q` nem calcular a chave privada `d` — a mensagem sai direto da
raiz.

---

## 2. Estrutura do código

### `raiz_inteira(x, n)`

Calcula a maior raiz inteira `r` tal que `r**n <= x`, **sem usar números
de ponto flutuante** (`float`). Isso é essencial: `x ** (1/n)` em Python
perde precisão para números muito grandes (centenas de dígitos), o que
faria o ataque falhar silenciosamente (um `round()` levemente errado nunca
bate com o valor exato).

A implementação usa **busca binária**:
1. Dobra um limite superior (`hi`) até ultrapassar `x` na potência `n`.
2. Faz busca binária entre `lo` e `hi` até encontrar o maior `r` que
   satisfaz `r**n <= x`.

Isso garante precisão exata, não importa o tamanho dos números.

### `main(limite_k=10_000_000)`

O laço principal do ataque:

1. Para cada `k` de 0 até o limite definido:
   - Calcula `alvo = c + k*N`.
   - Calcula a raiz e-ésima inteira de `alvo`.
   - Confere se essa raiz, elevada a `e`, realmente reproduz `alvo`
     exatamente (`r ** e == alvo`). Essa confirmação é o que evita falsos
     positivos da raiz aproximada.
2. Assim que encontrar, converte o valor em bytes e tenta decodificar
   como texto UTF-8.
3. Se não encontrar nenhum `k` dentro do limite, avisa para aumentar
   `limite_k` — mas normalmente, se a premissa do desafio estiver certa
   (M^e só um pouco maior que N), o `k` certo aparece bem no início da
   busca (valores pequenos, geralmente de 1 dígito).

---

## 3. Como usar

1. Abra o arquivo `values` do desafio e identifique `N`, `e`, `c`.
   Repare que aqui o `e` costuma ser pequeno (3, 5, 7...), diferente do
   RSA comum que usa `65537`.
2. Preencha as três variáveis no topo do script.
3. Rode: `python3 mini_rsa.py`.

---

## 4. Limitações e cuidados

- **Depende da premissa do desafio**: isso só funciona porque o enunciado
  garante que `M^e` é "só um pouco maior" que `N`. Se a mensagem tivesse
  padding robusto (como o OAEP usado no RSA real), `M^e` poderia ser
  ordens de magnitude maior que `N`, exigindo um `k` enorme — inviável de
  testar um por um.
- **Precisão dos números grandes**: por isso a função de raiz inteira
  evita float completamente. Qualquer tentativa de usar
  `round(alvo ** (1/e))` pode parecer funcionar às vezes e falhar em
  outras, de forma imprevisível — por isso o script sempre confirma com
  `r ** e == alvo` antes de aceitar o resultado.
- **Mensagem no meio de "lixo"**: alguns desafios colocam a flag
  cercada de texto de padding (nem sempre no início da mensagem
  decodificada). Se o `.decode()` falhar, olhe os bytes brutos impressos
  — a flag pode estar visível no meio deles.
