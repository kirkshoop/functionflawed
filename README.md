# wg21-starter

This template will setup a repo that uses https://github.com/mpark/wg21 to write wg21 (C++) papers
There is a CI action that produces a build artifact containg the generated pdf file(s).
`paper.md` is an empty paper. 

run `make update` to get latest paper references

run `make` to generate a pdf from the .md files

## Requirements

  - `python3`
  - `xelatex`
  - `plantuml`

### OS X

```bash
brew install --cask mactex
```

```bash
brew install plantuml
```

### Ubuntu

```bash
sudo apt-get install texlive-latex-base plantuml
```

### Debian

Debian installation may require these additional packages:

  - `texlive-fonts-recommended`
  - `texlive-latex-recommended`
  - `texlive-latex-extra`
