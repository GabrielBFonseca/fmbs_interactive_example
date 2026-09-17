# Configuração do Ambiente

Este projeto foi desenvolvido utilizando **Python 3.11**. Recomenda-se utilizar o **Miniconda** para criar e gerenciar o ambiente Python do projeto.

## 1. Instalar o Miniconda

Caso ainda não tenha o Miniconda instalado, faça a instalação de acordo com seu sistema operacional.

Após a instalação, verifique se o comando `conda` está disponível:

```bash
conda --version
```

## 2. Criar o ambiente

Abra um terminal na pasta do projeto e crie um novo ambiente chamado `fmbs` com Python 3.11:

```bash
conda create -n fmbs python=3.11
```

Quando solicitado, confirme a instalação.

Em seguida, ative o ambiente:

```bash
conda activate fmbs
```

Para verificar se a versão correta do Python está sendo utilizada:

```bash
python --version
```

O resultado deverá ser:

```text
Python 3.11.x
```

## 3. Instalar as dependências

Com o ambiente `fmbs` ativado, instale as dependências do projeto:

```bash
pip install -r requirements.txt
```

O arquivo `requirements.txt` contém as bibliotecas e versões necessárias para execução do projeto:

```text
numpy==2.4.6
scipy==1.17.1
matplotlib==3.11.1
opencv-contrib-python==5.0.0.93
scikit-image==0.26.0
imageio==2.37.4
pillow==12.3.0
networkx==3.6.1
higra==0.6.13
```

## 4. Instalar o Jupyter Notebook

Instale o Jupyter Notebook dentro do ambiente:

```bash
pip install notebook ipykernel
```

Registre o ambiente como um kernel do Jupyter:

```bash
python -m ipykernel install --user --name fmbs --display-name "Python 3.11 - fmbs"
```

## 5. Executar o projeto

Com o ambiente ativado:

```bash
conda activate fmbs
```

Inicie o Jupyter Notebook:

```bash
jupyter notebook
```

No Jupyter, abra o notebook desejado e, caso seja necessário selecionar manualmente o ambiente, escolha o kernel:

```text
Python 3.11 - fmbs
```

## Próximas execuções

Após realizar a configuração inicial, não será necessário reinstalar as dependências.

Para executar novamente o projeto, basta abrir um terminal na pasta do projeto e executar:

```bash
conda activate fmbs
jupyter notebook
```

Ao terminar, o ambiente pode ser desativado com:

```bash
conda deactivate
```
