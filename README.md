# SAB-AI-Scope-workshop

This repository contains materials for the SAB-AI-Scope workshop. The workshop is focused on introducing AI-Scope, an astronomical data platform designed to facilitate not only but especially machine learning and deep learning workflows in astronomy. The materials include presentations, documentation, and resources related to AI-Scope's features and capabilities.

## Preparação

Execute os notebooks na ordem `00`–`06`. O notebook `01` produz
`notebooks/outputs/cluster_1.csv`, usado por `02` e `03`. O notebook `04`
explora imagens; o `05` monta uma amostra de até 1.000 fontes com Gaia, DESI,
ZTF DR22 e cutouts FITS em `notebooks/outputs/multimodal/`; o `06` demonstra
um replay finito de alertas.
As consultas e os produtos de imagem exigem acesso à instância AI-Scope e
credenciais válidas. A cobertura de DESI e os alertas disponíveis podem mudar.

As referências locais da API e da linguagem de consulta estão em
[`docs/python_api.md`](docs/python_api.md) e
[`docs/query_examples.md`](docs/query_examples.md).
