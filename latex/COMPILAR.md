# Compilar arquivos TEX

## Requisitos

Ferramenta necessaria: [https://miktex.org/download](https://miktex.org/download)

**Adicionar ao path**

No windows, procure por "environment", e selecione "Edit Environment Variables for your account". Na seção "User variables", selecione a variável "Path" e clique em "Edit". Clique em "New" e adicione o caminho para a pasta onde o MiKTeX foi instalado, por exemplo: `C:\Program Files\MiKTeX 2.9\miktex\bin\x64`. Clique em "OK" para salvar as alterações.    

## Compilar arquivos .tex

Para compilar os arquivos .tex, abra o terminal na pasta onde estão os arquivos e execute o seguinte comando:

```sh
pdflatex.exe --output-directory=out Relatorio.tex
```

Caso queira compilar todos os arquivos que começam com "Rel", utilize o seguinte comando:

```sh
fd -d1 "Rel*" | while read a; do pdflatex.exe --output-directory=out $a done
```

