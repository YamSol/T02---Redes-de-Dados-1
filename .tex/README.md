# Relatórios em LaTeX — T02 Redes de Dados

Versão LaTeX dos Relatórios 10, 11 e 12. Estrutura pronta para impressão (A4, margens 2.5cm, capa com logo + título, sumário, cabeçalho/rodapé).

## Arquivos

- `preamble.tex` — preâmbulo compartilhado (pacotes, estilos, capa, caixas de nota, listings).
- `Relatorio_10.tex` — TLS em HTTP com Wireshark.
- `Relatorio_11.tex` — IDS com Suricata.
- `Relatorio_12.tex` — Firewall pfSense.
- `img/` — coloque aqui os arquivos de imagem locais (em especial `img/logo.png`).

## Compilar

Requer uma distribuição LaTeX (TeX Live, MiKTeX). Na pasta `.tex/`:

```bash
pdflatex Relatorio_10.tex && pdflatex Relatorio_10.tex
pdflatex Relatorio_11.tex && pdflatex Relatorio_11.tex
pdflatex Relatorio_12.tex && pdflatex Relatorio_12.tex
```

A segunda passada é necessária para o sumário ficar correto.

## Imagens

Os relatórios originais usam imagens hospedadas no GitHub (`user-attachments`), que **não podem** ser baixadas pelo `pdflatex` durante a compilação. O preâmbulo define o comando `\imgph{largura}{URL}` que renderiza uma **caixa-placeholder** com a URL embaixo — assim a compilação roda sem erro mesmo sem as imagens locais.

Para usar as imagens reais:
1. Baixe cada uma localmente para `.tex/img/` (ex.: `img/r10_natnetwork.png`).
2. Substitua a chamada `\imgph{...}{URL}` pelo `\includegraphics`:

```latex
\begin{figure}[H]\centering
  \includegraphics[width=0.7\linewidth]{img/r10_natnetwork.png}
\end{figure}
```

## Logo da capa

Coloque o arquivo em `.tex/img/logo.png`. Se não existir, a capa exibe um quadro vazio com `[logo.png]` no lugar — sem quebrar a compilação.
