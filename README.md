# Konfiguracja Ubuntu Server
Konfiguracja Ubuntu Server 24.04 pod INF.02
## Budowanie:
Uzyj narzędzia [pandoc](https://github.com/jgm/pandoc) i [template'u](https://github.com/Wandmalfarbe/pandoc-latex-template).
Komenda na linuxie: 
```sh
pandoc ubuntu\ server\ 24.04.md -o ubuntu\ server\ 24.04.pdf --from markdown --template eisvogel -V listings=false -V minted=true -V lang=pl --pdf-engine=xelatex --highlight-style=tango
```
Narzędzia znikneły? użyj xelatex'a i pliku .tex
```sh
xelatex ubuntu\ server\ 24.04.tex
```
## PDF dostępny w [releases](https://github.com/TexturedPolak/konfiguracja-ubuntu-server/releases)
