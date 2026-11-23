# Instruções e comandos de execução dos programas serial e paralelo no Cluster do CCAD/CEFET-MG

Documentação de apoio ao Projeto Final de Curso (PFC) em Engenharia Elétrica do **CEFET-MG**, intitulado *"Modelagem computacional de dispositivos eletromagnéticos utilizando processamento de alto desempenho"* (Autor: Rennan William Andrade Santos | Orientador: Prof. Dr. Eduardo Henrique da Rocha Coppoli).

Este guia apresenta o fluxo completo de transferência de arquivos, compilação, submissão via SLURM e resgate de resultados para simulações de um atuador eletromagnético sob movimento, empregando o método sem malha **LRPIM** (*Local Radial Point Interpolation Method*) nas versões sequencial em CPU (**Fortran 95**) e paralela em GPU (**CUDA Fortran**), executadas no cluster do **Centro de Computação de Alto Desempenho do CEFET-MG (CCAD/CEFET-MG)**.

---

## 1. Pré-requisitos de Acesso

1. **Conta de acesso ativa no CCAD** (solicite via chamado na Central de Serviços de TI: https://cs.sgi.cefetmg.br/);
2. **VPN do CEFET-MG ativa** (aplicativo FortiClient VPN, disponível em https://fortinet.com/support/product-downloads/);
3. **Cliente SSH instalado** (nativo no Linux/macOS; no Windows, use o Terminal, PowerShell ou PuTTY).

---

## 2. Conexão e Transferência de Arquivos

Com a VPN conectada, abra o terminal local e siga os passos:

### Testar conectividade e acessar o nó de login
```bash
ping login.ccad.cefetmg.br
ssh <SEU_CPF>@login.ccad.cefetmg.br
```

### Criar os diretórios de trabalho no cluster
```bash
mkdir -p ~/atuador/paralelo
mkdir -p ~/atuador/serial
```

### Enviar os arquivos do computador local para o cluster (`scp`)

**Para a versão paralela (GPU):**
```bash
scp main.cuf variableslocal.h Input11011.txt <SEU_CPF>@login.ccad.cefetmg.br:~/atuador/paralelo/
```

**Para a versão serial (CPU):**
```bash
scp main.f95 variableslocal.h Input11011.txt <SEU_CPF>@login.ccad.cefetmg.br:~/atuador/serial/
```

---

## 3. Compilação no Nó de Login

> **Atenção:** O nó de login (`login.ccad.cefetmg.br`) deve ser usado **apenas** para transferência de arquivos, compilação e submissão de jobs. Nunca execute simulações diretamente nele.

### Compilação do programa paralelo
```bash
cd ~/atuador/paralelo

module purge
module load gnu15
module load cuda/25.1
module load nvhpc-hpcx-cuda12/25.1
export LD_LIBRARY_PATH=/opt/ohpc/pub/compiler/gcc/15.2.0/lib64:$LD_LIBRARY_PATH
export LIBRARY_PATH=/opt/ohpc/pub/compiler/gcc/15.2.0/lib64:$LIBRARY_PATH

nvfortran -cuda -O2 -o atuadorFinal main.cuf -lcusolver -lcublas
```

### Compilação do programa serial
```bash
cd ~/atuador/serial

module purge
module load gnu15

gfortran -O2 main.f95 -o atuadorSerial
```

---

## 4. Scripts de Submissão (SLURM)

### Script Paralelo — GPU

No diretório `~/atuador/paralelo`, crie o arquivo com `nano job_gpu.slurm` e insira o conteúdo abaixo:

```bash
#!/bin/bash
#SBATCH --job-name=teste_atuador
#SBATCH --partition=gpu
#SBATCH --qos=gpu_qos
#SBATCH --constraint=ssd
#SBATCH --nodes=1
#SBATCH --ntasks-per-node=1
#SBATCH --gres=gpu:1
#SBATCH --mem=32G
#SBATCH --time=01-00:00:00
#SBATCH --output=resultado_%j.out
#SBATCH --error=erros_%j.err

# Limpar e carregar os mesmos módulos da compilação
module purge
module load gnu15 cuda/25.1 nvhpc-hpcx-cuda12/25.1

# Exportar os caminhos das bibliotecas
export LD_LIBRARY_PATH=/opt/ohpc/pub/compiler/gcc/15.2.0/lib64:$LD_LIBRARY_PATH
export LIBRARY_PATH=/opt/ohpc/pub/compiler/gcc/15.2.0/lib64:$LIBRARY_PATH

# Aumentar limite de pilha da CPU para evitar Segmentation Fault com cuSOLVER
ulimit -s unlimited

# Conceder permissão de execução ao binário na pasta original
chmod +x $SLURM_SUBMIT_DIR/atuadorFinal

# 1. Configurar e criar o diretório temporário no /scratch local em SSD
SCRATCH_DIR=/scratch/$USER/$SLURM_JOB_ID
mkdir -p $SCRATCH_DIR

# 2. Copiar o executável e o arquivo de entrada para o /scratch
cp $SLURM_SUBMIT_DIR/atuadorFinal $SCRATCH_DIR/
cp $SLURM_SUBMIT_DIR/Input11011.txt $SCRATCH_DIR/

# 3. Entrar no /scratch para realizar a execução I/O intensiva lá dentro
cd $SCRATCH_DIR

# 4. Executar o programa medindo o tempo exato com o comando 'time'
time ./atuadorFinal

# 5. Criar a pasta final (caso não exista) e copiar os resultados de volta para o /home
DIRETORIO_FINAL=~/pfc_rennan/programaParalelo
mkdir -p $DIRETORIO_FINAL
cp *.bin $DIRETORIO_FINAL/
```

### Script Serial — CPU

No diretório `~/atuador/serial`, crie o arquivo com `nano job_serial.slurm` e insira o conteúdo abaixo:

```bash
#!/bin/bash
#SBATCH --job-name=serial_atuador
#SBATCH --partition=dev
#SBATCH --qos=dev_qos
#SBATCH --constraint=ssd
#SBATCH --nodes=1
#SBATCH --ntasks-per-node=1
#SBATCH --mem=32G
#SBATCH --time=04:00:00
#SBATCH --output=resultado_%j.out
#SBATCH --error=erros_%j.err

# Limpar e carregar apenas o módulo necessário para execução
module purge
module load gnu15

# Navegar para o diretório de submissão
cd $SLURM_SUBMIT_DIR

# Conceder permissão de execução
chmod +x ./atuadorSerial

# 1. Configurar e criar o diretório temporário no /scratch local em SSD
SCRATCH_DIR=/scratch/$USER/$SLURM_JOB_ID
mkdir -p $SCRATCH_DIR

# 2. Copiar o executável e arquivo de entrada para o /scratch
cp ./atuadorSerial $SCRATCH_DIR/
cp ./Input11011.txt $SCRATCH_DIR/

# 3. Entrar no /scratch
cd $SCRATCH_DIR

# 4. Executar medindo o tempo
time ./atuadorSerial

# 5. Voltar os resultados para a sua pasta no /home
cp *.bin $SLURM_SUBMIT_DIR/
```

---

## 5. Submissão e Monitoramento via SLURM

### Submeter o job para a fila de computação
```bash
# Para GPU:
sbatch job_gpu.slurm

# Para CPU:
sbatch job_serial.slurm
```

### Acompanhar status e métricas
```bash
# Verificar jobs ativos na fila
squeue -u $USER

# Consultar histórico e tempo decorrido após conclusão (substitua <JOB_ID> pelo número do job)
sacct -j <JOB_ID> --format=JobID,JobName,Elapsed,State,ExitCode

# Ler saída padrão e tempo medido pelo comando 'time'
cat resultado_<JOB_ID>.out
cat erros_<JOB_ID>.err
```

> **Nota sobre I/O:** Os scripts copiam o executável e o arquivo de entrada para o diretório temporário `/scratch/$USER/$SLURM_JOB_ID` (em disco SSD) durante o cálculo e retornam os arquivos `.bin` para o `/home` ao final da execução.

---

## 6. Resgate dos Resultados para Pós-Processamento

Após desconectar do cluster (`exit`), baixe os arquivos `.bin` gerados para a sua máquina local para análise no MATLAB:

```bash
# Resultados da execução paralela
scp <SEU_CPF>@login.ccad.cefetmg.br:~/atuador/paralelo/*.bin .

# Resultados da execução serial
scp <SEU_CPF>@login.ccad.cefetmg.br:~/atuador/serial/*.bin .
```

---

## Agradecimentos

*O autor agradece ao Centro de Computação de Alto Desempenho do CEFET-MG (CCAD/CEFET-MG) pelos recursos computacionais disponibilizados para a realização deste Projeto Final de Curso.*
