# Como o som do "tique" foi gerado

Não veio de nenhuma lib ou sample pronto — é síntese de áudio procedural
(bem comum em áudio de jogos), feita com Web Audio API puro, sem
nenhum arquivo de áudio externo. A receita é ruído + envelope + filtro.

## 1. Ruído branco como matéria-prima

```js
dados[i] = (Math.random()*2 - 1) * Math.pow(1 - i/n, 2);
```

`Math.random()*2-1` gera um valor aleatório entre -1 e 1 para cada
amostra do buffer — isso é ruído branco puro (todas as frequências
presentes, sem tom definido). Um clique real (peça de plástico/metal
batendo) também não tem uma nota musical — é uma explosão de energia
em muitas frequências, então ruído é a base certa. Um oscilador
(que gera um tom puro, tipo um "beep") soaria eletrônico demais para
imitar um pino físico.

## 2. Envelope de decaimento — o que dá forma de "tique" e não de "chiado"

O `Math.pow(1 - i/n, 2)` multiplica cada amostra por um valor que
começa em 1 e cai pra 0 ao longo do buffer (25ms) — um envelope
percussivo clássico: começa forte, decai rápido. Sem isso seria só
ruído constante (tipo estática de TV). É o mesmo princípio de um
envelope ADSR usado em qualquer synth, só que aplicado direto na
forma de onda. Há também um segundo envelope redundante no `GainNode`
(`exponentialRampToValueAtTime`), que reforça esse decaimento.

## 3. Filtro passa-alta — o que dá o timbre "seco e metálico"

```js
filtro.type = 'highpass';
filtro.frequency.value = 2200;
```

Corta tudo abaixo de 2200Hz, deixando só as frequências agudas do
ruído. Isso diferencia um "tique" (agudo, seco) de um "bum" (grave,
surdo) — mesma matéria-prima (ruído), timbre completamente diferente
só pela filtragem.

Essa combinação (ruído + envelope + filtro) é basicamente o algoritmo
por trás de percussão sintética em bibliotecas de áudio de jogos (tipo
o clássico "hi-hat" ou "snare" sintetizado da era 8-bit/chiptune).

---

# Como a roleta sabe a hora de bater o último pino / parar

Não existe nenhuma detecção de "último pino". O resultado é decidido
matematicamente **antes** de a roleta começar a girar, e a animação
inteira é calculada, do primeiro ao último frame, para terminar ali.

## 1. O sorteio acontece primeiro, a roleta só ilustra depois

```js
elSpin.addEventListener('click', function(){
  var v = sortear();          // decide o vencedor JÁ, antes de qualquer giro
  ...
  wheel.spinToItem(v, ...);   // só pede pra roleta "ir visualmente até o índice v"
});
```

`sortear()` roda o `Math.random()` ponderado pelos pesos uma única
vez, no clique. A roleta gira depois disso só para *encenar* um
resultado que já está decidido.

## 2. `spinToItem` calcula o ângulo final matematicamente, não por tentativa

Dentro da lib (`spinToItem`), ela pega o índice vencedor, calcula o
ângulo daquela fatia, soma as voltas extras (6 revoluções, para o
efeito dramático) e chega numa **rotação-alvo final exata** — um
número fixo, definido antes do primeiro frame de animação. Não existe
um loop tipo "gira até acertar"; é: "rotação atual → rotação final X,
em 5200ms".

## 3. A cada frame, ela interpola entre início e fim usando o easing

```js
rotation = inicio + (fim - inicio) * easeOutQuint(progresso)
```

`easeOutQuint` é uma curva que começa rápida e desacelera suavemente
até 0 no final — é isso que faz a roleta parecer desacelerar de
verdade, e não nenhum código dizendo "estamos perto, vai devagar".

## 4. O tique é só uma consequência dessa interpolação

O callback `onCurrentIndexChange: tocarTique` dispara toda vez que o
ângulo calculado no passo 3 cruza para uma fatia diferente. Como a
curva de easing desacelera, os cruzamentos (e os tiques) ficam
naturalmente mais espaçados perto do fim — sem nenhum cálculo de
"quando bater o último pino". Ele só bate o último quando a
interpolação, matematicamente, entra na fatia vencedora — que é
exatamente onde ela vai parar.

## 5. Quando o tempo acaba, ela "cola" no valor exato e avisa

```js
if (e >= this._spinToTimeEnd){
  this.rotation = this._spinToEndRotation; // snap exato, sem arredondamento visual
  this.raiseEvent_onRest();                // dispara o onRest → mostra o resultado
}
```

Resumo: a "sorte" é decidida instantaneamente no clique; a animação
inteira (incluindo os tiques) é só a curva matemática de um valor A
até um valor B pré-calculado — nada é decidido "em tempo real"
enquanto ela gira.
