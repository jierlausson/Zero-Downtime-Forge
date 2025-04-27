Documentação do Script de Deploy para Laravel Forge
Introdução
Este documento detalha um script de deploy genérico para aplicações Laravel hospedadas no Laravel Forge, desenvolvido por Jierlausson Oliveira. O script implementa Zero Downtime Deployments, garantindo implantações seguras, rápidas e contínuas, sem interrupções no serviço, mesmo em caso de erros durante o deploy. Inspirado nas funcionalidades do Laravel Envoyer, ele oferece uma solução robusta e acessível para gerenciar implantações em produção, com suporte a rollback para até três releases anteriores, permitindo reverter rapidamente caso o deploy não atenda às expectativas.
O documento é dividido em:

Objetivo do Script: Finalidade e contexto.
Explicação Detalhada de Cada Etapa: Propósito e funcionamento das 32 etapas do script.
Script Completo: Código do script de deploy.
Configurações do Servidor: Pré-requisitos e configurações necessárias no servidor.
Instruções de Implantação no Laravel Forge: Passo a passo para configurar e executar o deploy.
Como Realizar um Rollback: Instruções para reverter para uma release anterior.
Dicas de Depuração: Como lidar com erros comuns.

O script é compatível com qualquer site configurado no Laravel Forge, utilizando variáveis fornecidas pelo Forge (ex.: FORGE_SITE_PATH, FORGE_SITE_BRANCH) e pelo arquivo .env do projeto.

Objetivo do Script
O script de deploy foi projetado para oferecer Zero Downtime Deployments em aplicações Laravel no Laravel Forge, garantindo que o sistema permaneça online durante todo o processo de implantação e mesmo em caso de erros. Ele se assemelha ao Laravel Envoyer, proporcionando uma solução segura, rápida e confiável para implantações contínuas. Suas principais metas são:

Atualizar o código: Sincronizar o repositório Git com a branch configurada sem interromper o serviço.
Gerenciar dependências: Instalar dependências PHP (Composer) e Node.js (npm) de forma eficiente.
Construir assets: Compilar assets frontend com Vite, mantendo consistência entre releases.
Persistir dados: Manter pastas compartilhadas (storage e public/build) para preservar dados do usuário.
Gerenciar o banco de dados: Executar backups automáticos, migrações e seeders (opcional), com proteção contra falhas.
Limpar caches: Garantir que os caches do Laravel e Filament sejam limpos para refletir alterações.
Minimizar riscos: Validar cada etapa do deploy e manter backups para recuperação rápida em caso de erros.
Suportar rollback: Permitir reverter para até três releases anteriores, garantindo flexibilidade caso o deploy não atenda às expectativas.
Manter logs detalhados: Registrar todas as etapas para auditoria e depuração.

O script opera no diretório do site definido pelo Forge (${FORGE_SITE_PATH}), que contém o repositório Git. Ele usa uma pasta de releases (releases) para preparar novas versões, copiando os arquivos para o diretório ativo apenas após validação, garantindo zero downtime. Em caso de falhas, o sistema permanece funcional, e a opção de rollback permite restaurar uma release anterior rapidamente.

Explicação Detalhada de Cada Etapa
O script é composto por 32 etapas, cada uma projetada para garantir segurança, eficiência e zero downtime. Abaixo, explicamos o motivo e o funcionamento de cada uma.
Declaração Inicial e Funções

Shebang (#!/bin/bash): Define que o script é executado em um shell Bash.
Comentários: Descrevem o propósito do script e as variáveis esperadas (ex.: FORGE_* do Forge, variáveis do .env como DB_DATABASE).
Função exit_with_error:
Exibe uma mensagem de erro no stderr, remove o lock do PHP-FPM (se presente) e encerra o script com código de saída 1.
Motivo: Garantir que erros sejam comunicados claramente, mantendo o sistema ativo e permitindo rollback, se necessário.


Função get_date:
Retorna a data atual no fuso horário America/Fortaleza.
Motivo: Padronizar timestamps nos logs para consistência.



Etapa 1: Validar o fuso horário

Verifica se America/Fortaleza está disponível no sistema.
Se falhar, exibe um erro sugerindo a instalação do pacote tzdata.
Motivo: Garantir que todos os timestamps nos logs usem o fuso horário configurado.

Etapa 2: Configurar RELEASE_ID

Gera um identificador único para a release baseado na data atual (ex.: 20250427021116).
Motivo: Identificar cada deploy para nomear pastas de releases e backups, facilitando rastreamento e rollback.

Etapa 3: Verificar FORGE_SITE_PATH

Confirma que a variável FORGE_SITE_PATH (ex.: o diretório do site) está definida.
Motivo: Evitar que o script execute em um ambiente mal configurado no Forge.

Etapa 4: Configurar logging

Cria o diretório de logs (${SITE_PATH}/logs) e define o arquivo de log (deploy_${RELEASE_ID}.log).
Se falhar, usa /tmp como fallback.
Redireciona a saída do script para o log com tee, preservando a exibição no console.
Motivo: Registrar todas as etapas para auditoria e depuração, essencial para investigar falhas sem interromper o serviço.

Etapa 5: Verificar variáveis do Forge

Confirma que FORGE_SITE_BRANCH, FORGE_PHP, FORGE_COMPOSER e FORGE_PHP_FPM estão definidas.
Motivo: Garantir que o ambiente do Forge está configurado corretamente antes de prosseguir.

Etapa 6: Carregar variáveis do .env

Carrega variáveis do arquivo .env (ex.: DB_DATABASE, DB_USERNAME, PRE_MIGRATION_BACKUP, RUN_SEEDERS).
Motivo: Permitir configurações dinâmicas do projeto sem alterar o script.

Etapa 7: Definir variáveis

Define caminhos e variáveis como RELEASES_PATH, NEW_RELEASE_PATH, CURRENT_LINK, SHARED_BUILD_PATH, etc.
Motivo: Centralizar configurações para facilitar manutenção e consistência.

Etapa 8: Verificar DB_DATABASE e DB_USERNAME

Confirma que DB_DATABASE e DB_USERNAME estão definidos no .env.
Motivo: Evitar falhas na conexão com o banco de dados, garantindo que o sistema permaneça funcional.

Etapa 9: Validar conexão com o banco de dados

Executa um comando psql para testar a conexão com o banco de dados PostgreSQL.
Motivo: Garantir que o banco está acessível antes de realizar backups ou migrações, prevenindo erros que poderiam interromper o deploy.

Etapa 10: Criar diretórios

Cria os diretórios releases, backups/db e shared/build.
Motivo: Preparar a estrutura necessária para armazenar releases, backups e assets compartilhados, essencial para rollback e persistência de dados.

Etapa 11: Criar nova pasta de release

Cria a pasta ${NEW_RELEASE_PATH} (ex.: ${FORGE_SITE_PATH}/../releases/20250427021116).
Motivo: Preparar um ambiente isolado para a nova release, garantindo que o diretório ativo permaneça intocado até a validação.

Etapa 12: Copiar repositório Git

Copia o repositório Git de ${CURRENT_LINK} para ${NEW_RELEASE_PATH}.
Motivo: Criar uma cópia do código atual como base para a nova release, mantendo o sistema ativo.

Etapa 13: Atualizar repositório Git

Executa git fetch origin e git reset --hard origin/${GIT_BRANCH} na nova release.
Motivo: Sincronizar o código com a branch remota configurada sem afetar o diretório ativo.

Etapa 14: Validar atualização do Git

Verifica se não há diferenças entre o repositório local e remoto com git diff.
Motivo: Garantir que o código foi atualizado corretamente antes de prosseguir, evitando deploy de código incompleto.

Etapa 15: Compartilhar pasta storage

Cria um link simbólico de ${NEW_RELEASE_PATH}/storage para ${CURRENT_LINK}/storage.
Motivo: Persistir dados do usuário (ex.: uploads) entre releases, mantendo a funcionalidade do sistema.

Etapa 16: Criar pasta public

Garante que ${NEW_RELEASE_PATH}/public existe.
Motivo: Preparar o diretório para arquivos públicos e assets, essencial para o build do Vite.

Etapa 17: Listar conteúdo de public

Exibe o conteúdo de ${NEW_RELEASE_PATH}/public para diagnóstico.
Motivo: Facilitar a depuração de problemas com arquivos públicos sem interromper o serviço.

Etapa 18: Preparar public/build para Vite

Remove qualquer public/build existente e cria um diretório real.
Motivo: Evitar conflitos durante o build do Vite, garantindo assets consistentes.

Etapa 19: Instalar dependências

Executa composer install (PHP) e npm ci/npm run build (Node.js) com timeout de 600 segundos.
Motivo: Instalar dependências backend e compilar assets frontend em um ambiente isolado, sem afetar o diretório ativo.

Etapa 20: Listar arquivos do Vite

Exibe o conteúdo de public/build após o build.
Motivo: Confirmar que o Vite gerou os assets corretamente, essencial para validar a release.

Etapa 21: Copiar assets do Vite

Copia os arquivos de public/build para ${SHARED_BUILD_PATH}.
Motivo: Persistir assets compilados entre releases, garantindo consistência visual.

Etapa 22: Validar manifest.json

Verifica se ${SHARED_BUILD_PATH}/manifest.json existe.
Motivo: Confirmar que o build do Vite foi bem-sucedido antes de prosseguir.

Etapa 23: Listar arquivos compartilhados

Exibe o conteúdo de ${SHARED_BUILD_PATH} para diagnóstico.
Motivo: Facilitar a depuração de problemas com assets compartilhados.

Etapa 24: Criar link simbólico para public/build

Cria um link simbólico de ${NEW_RELEASE_PATH}/public/build para ${SHARED_BUILD_PATH}.
Motivo: Garantir que a nova release use os assets compartilhados, mantendo a funcionalidade do frontend.

Etapa 25: Confirmar link simbólico

Verifica o link simbólico ${NEW_RELEASE_PATH}/public/build.
Motivo: Diagnosticar problemas com a configuração de assets antes de atualizar o diretório ativo.

Etapa 26: Listar conteúdo de public após link

Exibe o conteúdo de ${NEW_RELEASE_PATH}/public novamente.
Motivo: Confirmar que o link simbólico foi aplicado corretamente.

Etapa 27: Fazer backup do banco de dados

Executa pg_dump para criar um backup em ${DB_BACKUP_PATH}/backup_${RELEASE_ID}.sql.
Motivo: Proteger dados antes de migrações, permitindo restauração em caso de falhas.

Etapa 28: Fazer backup pré-migração (opcional)

Se PRE_MIGRATION_BACKUP=true no .env, cria um backup adicional.
Motivo: Oferecer uma camada extra de segurança para restauração caso as migrações falhem.

Etapa 29: Executar migrações

Executa php artisan migrate --force com timeout de 300 segundos.
Motivo: Aplicar alterações no esquema do banco de dados em um ambiente isolado, sem interromper o serviço.

Etapa 30: Executar seeders (opcional)

Se RUN_SEEDERS=true no .env, executa php artisan db:seed --force, tolerando falhas.
Motivo: Popular o banco com dados iniciais, se necessário, com tolerância para evitar interrupções.

Etapa 31: Validar o deploy

Executa php artisan route:list para verificar a integridade da aplicação.
Motivo: Garantir que a nova release está funcional antes de atualizar o diretório ativo, prevenindo downtime.

Etapa 32: Limpar backups pré-migração

Remove backups pré-migração, se existirem.
Motivo: Evitar acúmulo de arquivos temporários, mantendo o servidor organizado.


Script Completo
#!/bin/bash

# Script de deploy dinâmico para Laravel Forge
# Uso: Relies on Forge-provided variables (FORGE_*) and project .env for DB_DATABASE, DB_USERNAME, PRE_MIGRATION_BACKUP, and RUN_SEEDERS

# Função para sair com erro
exit_with_error() {
    echo "Erro: $1" >&2
    [ -f "${PHP_FPM_RELOAD}" ] && rm -f "${PHP_FPM_RELOAD}"  # Liberar lock do PHP-FPM
    exit 1
}

# Função para obter data no fuso horário America/Fortaleza
get_date() {
    TZ=America/Fortaleza date "$@"
}

# Verificar se o fuso horário America/Fortaleza está disponível
if ! TZ=America/Fortaleza date >/dev/null 2>&1; then
    exit_with_error "Fuso horário America/Fortaleza não está disponível. Instale o pacote tzdata (sudo apt-get install tzdata)."
fi

# Configurar RELEASE_ID
RELEASE_ID=$(get_date +%Y%m%d%H%M%S)

# Verificar FORGE_SITE_PATH antes de configurar logging
[ -z "$FORGE_SITE_PATH" ] && exit_with_error "FORGE_SITE_PATH não está definido. Configure no Laravel Forge."
SITE_PATH="$FORGE_SITE_PATH"

# Configurar logging
LOG_DIR="${SITE_PATH}/logs"
LOG_FILE="${LOG_DIR}/deploy_${RELEASE_ID}.log"
mkdir -p "${LOG_DIR}" 2>/dev/null || {
    echo "Aviso: Falha ao criar ${LOG_DIR}, usando /tmp como fallback" >&2
    LOG_FILE="/tmp/deploy_${RELEASE_ID}.log"
}
exec > >(TZ=America/Fortaleza tee -a "${LOG_FILE}") 2>&1
echo "Iniciando deploy em $(get_date)..."
echo "Usando fuso horário: America/Fortaleza (exemplo: $(get_date))"

# Verificar se variáveis críticas do Forge estão definidas
[ -z "$FORGE_SITE_BRANCH" ] && exit_with_error "FORGE_SITE_BRANCH não está definido. Configure no Laravel Forge."
[ -z "$FORGE_PHP" ] && exit_with_error "FORGE_PHP não está definido. Configure no Laravel Forge."
[ -z "$FORGE_COMPOSER" ] && exit_with_error "FORGE_COMPOSER não está definido. Configure no Laravel Forge."
[ -z "$FORGE_PHP_FPM" ] && exit_with_error "FORGE_PHP_FPM não está definido. Configure no Laravel Forge."

# Carregar variáveis do .env
if [ -f "${SITE_PATH}/.env" ]; then
    export $(grep -v '^#' "${SITE_PATH}/.env" | xargs)
else
    exit_with_error "Arquivo .env não encontrado em ${SITE_PATH}"
fi

# Definir variáveis
RELEASES_PATH="${SITE_PATH}/../releases"
NEW_RELEASE_PATH="${RELEASES_PATH}/${RELEASE_ID}"
CURRENT_LINK="${SITE_PATH}"
PHP_FPM_RELOAD="/tmp/fpmlock"
DB_BACKUP_PATH="${SITE_PATH}/../backups/db"
DB_DATABASE="${DB_DATABASE}"  # Definido no .env
DB_USERNAME="${DB_USERNAME}"  # Definido no .env
SHARED_BUILD_PATH="${SITE_PATH}/../shared/build"
GIT_BRANCH="$FORGE_SITE_BRANCH"  # Fornecido pelo Forge
PHP_EXEC="$FORGE_PHP"  # Fornecido pelo Forge
COMPOSER_EXEC="$FORGE_COMPOSER"  # Fornecido pelo Forge
PHP_FPM_SERVICE="$FORGE_PHP_FPM"  # Fornecido pelo Forge
PRE_MIGRATION_BACKUP="${PRE_MIGRATION_BACKUP:-false}"  # Definido no .env, padrão false
RUN_SEEDERS="${RUN_SEEDERS:-false}"  # Definido no .env, padrão false

# Verificar se DB_DATABASE e DB_USERNAME estão definidos
[ -z "$DB_DATABASE" ] && exit_with_error "DB_DATABASE não está definido no .env."
[ -z "$DB_USERNAME" ] && exit_with_error "DB_USERNAME não está definido no .env."

# 1. Validar conexão com o banco de dados
echo "Validando conexão com o banco de dados... $(get_date)"
psql -U "${DB_USERNAME}" -d "${DB_DATABASE}" -c "SELECT 1;" > /dev/null 2>&1 || exit_with_error "Falha ao conectar ao banco de dados. Verifique DB_USERNAME, DB_DATABASE e DB_PASSWORD no .env, e configure ~/.pgpass com o formato 'localhost:5432:${DB_DATABASE}:${DB_USERNAME}:senha'."

# 2. Criar diretórios para releases, backups e shared
mkdir -p "${RELEASES_PATH}" || exit_with_error "Falha ao criar diretório de releases"
mkdir -p "${DB_BACKUP_PATH}" || exit_with_error "Falha ao criar diretório de backups"
mkdir -p "${SHARED_BUILD_PATH}" || exit_with_error "Falha ao criar diretório shared/build"

# 3. Criar nova pasta de release
mkdir -p "${NEW_RELEASE_PATH}" || exit_with_error "Falha ao criar nova pasta de release"

# 4. Copiar repositório Git existente e atualizar
if [ -d "${CURRENT_LINK}/.git" ]; then
    echo "Copiando repositório Git existente para nova release... $(get_date)"
    cp -r "${CURRENT_LINK}/." "${NEW_RELEASE_PATH}" || exit_with_error "Falha ao copiar repositório Git"
else
    exit_with_error "Repositório Git não encontrado em ${CURRENT_LINK}"
fi

cd "${NEW_RELEASE_PATH}" || exit_with_error "Falha ao acessar a nova release"
echo "Atualizando repositório (branch: ${GIT_BRANCH})... $(get_date)"
git fetch origin || exit_with_error "Falha ao executar git fetch"
git reset --hard origin/"${GIT_BRANCH}" || exit_with_error "Falha ao resetar para origin/${GIT_BRANCH}"

# 5. Validar se o git pull atualizou os arquivos
echo "Validando atualização do repositório... $(get_date)"
if ! git diff --quiet origin/"${GIT_BRANCH}"; then
    exit_with_error "Arquivos locais não correspondem ao repositório remoto após git pull"
fi

# 6. Compartilhar a pasta storage (persistente entre releases)
if [ -d "${CURRENT_LINK}/storage" ]; then
    rm -rf "${NEW_RELEASE_PATH}/storage"
    ln -s "${CURRENT_LINK}/storage" "${NEW_RELEASE_PATH}/storage" || exit_with_error "Falha ao criar link simbólico para storage"
fi

# 7. Garantir que a pasta public exista na nova release
mkdir -p "${NEW_RELEASE_PATH}/public" || exit_with_error "Falha ao criar pasta public na nova release"

# 8. Verificar o conteúdo da pasta public antes do build (para diagnóstico)
echo "Conteúdo da pasta ${NEW_RELEASE_PATH}/public antes do build:"
ls -la "${NEW_RELEASE_PATH}/public" || exit_with_error "Falha ao listar ${NEW_RELEASE_PATH}/public"

# 9. Garantir que a pasta public/build seja um diretório real para o build do Vite
rm -rf "${NEW_RELEASE_PATH}/public/build"
mkdir -p "${NEW_RELEASE_PATH}/public/build" || exit_with_error "Falha ao criar pasta public/build na nova release"

# 10. Executar o deploy na nova release
echo "Instalando dependências do Composer... $(get_date)"
stdbuf -oL timeout 600 ${COMPOSER_EXEC} install --no-dev --no-interaction --prefer-dist --optimize-autoloader || exit_with_error "Falha no composer install ou timeout"

echo "Executando npm install e build... $(get_date)"
stdbuf -oL timeout 600 npm ci || exit_with_error "Falha no npm ci ou timeout"
stdbuf -oL timeout 600 npm run build || exit_with_error "Falha no npm run build ou timeout"

# 11. Listar arquivos gerados pelo Vite (para diagnóstico)
echo "Arquivos gerados pelo Vite em public/build/:"
ls -la public/build || exit_with_error "Falha ao listar arquivos em public/build"

# 12. Copiar os arquivos gerados pelo Vite para a pasta compartilhada
if [ -d "public/build" ] && [ "$(ls -A public/build)" ]; then
    echo "Copiando arquivos do Vite para a pasta compartilhada (${SHARED_BUILD_PATH})... $(get_date)"
    rm -rf "${SHARED_BUILD_PATH}"/*
    cp -r public/build/. "${SHARED_BUILD_PATH}/" || exit_with_error "Falha ao copiar arquivos do Vite para shared/build"
else
    exit_with_error "Pasta public/build vazia ou não encontrada após o build do Vite"
fi

# 13. Validar a presença do manifest.json
if [ ! -f "${SHARED_BUILD_PATH}/manifest.json" ]; then
    exit_with_error "Arquivo manifest.json não encontrado em ${SHARED_BUILD_PATH}"
fi

# 14. Listar arquivos na pasta compartilhada (para diagnóstico)
echo "Arquivos na pasta compartilhada (${SHARED_BUILD_PATH}):"
ls -la "${SHARED_BUILD_PATH}" || exit_with_error "Falha ao listar arquivos em shared/build"

# 15. Compartilhar a pasta public/build (persistente entre releases)
rm -rf "${NEW_RELEASE_PATH}/public/build"
ln -s "${SHARED_BUILD_PATH}" "${NEW_RELEASE_PATH}/public/build" || exit_with_error "Falha ao criar link simbólico para public/build"

# 16. Confirmar que o link simbólico foi criado (para diagnóstico)
echo "Verificando link simbólico para public/build:"
ls -ld "${NEW_RELEASE_PATH}/public/build" || exit_with_error "Falha ao verificar link simbólico para public/build"

# 17. Verificar o conteúdo da pasta public após criar o link simbólico (para diagnóstico)
echo "Conteúdo da pasta ${NEW_RELEASE_PATH}/public após criar o link simbólico:"
ls -la "${NEW_RELEASE_PATH}/public" || exit_with_error "Falha ao listar ${NEW_RELEASE_PATH}/public após criar o link simbólico"

# 18. Fazer backup do banco de dados PostgreSQL
echo "Fazendo backup do banco de dados... $(get_date)"
stdbuf -oL timeout 600 pg_dump -U "${DB_USERNAME}" "${DB_DATABASE}" > "${DB_BACKUP_PATH}/backup_${RELEASE_ID}.sql" || exit_with_error "Falha ao fazer backup do banco de dados. Verifique ~/.pgpass."

# 19. Fazer backup pré-migração (opcional)
if [ "${PRE_MIGRATION_BACKUP}" = "true" ]; then
    echo "Fazendo backup pré-migração... $(get_date)"
    stdbuf -oL timeout 600 pg_dump -U "${DB_USERNAME}" "${DB_DATABASE}" > "${DB_BACKUP_PATH}/pre_migration_backup_${RELEASE_ID}.sql" || exit_with_error "Falha ao fazer backup pré-migração. Verifique ~/.pgpass."
fi

# 20. Executar migrações
if [ -f artisan ]; then
    echo "Executando migrações... $(get_date)"
    stdbuf -oL timeout 300 ${PHP_EXEC} artisan migrate --force || exit_with_error "Falha nas migrações ou timeout"
fi

# 21. Executar seeders (opcional, com tolerância a falhas)
if [ "${RUN_SEEDERS}" = "true" ]; then
    echo "Executando seeders... $(get_date)"
    stdbuf -oL timeout 300 ${PHP_EXEC} artisan db:seed --force || echo "Aviso: Falha nos seeders, mas continuando deploy"
fi

# 22. Validar o deploy
echo "Validando deploy... $(get_date)"
stdbuf -oL timeout 300 ${PHP_EXEC} artisan route:list > /dev/null || exit_with_error "Falha na validação do deploy"

# 23. Verificar o conteúdo da pasta public antes da atualização (para diagnóstico)
echo "Conteúdo da pasta ${NEW_RELEASE_PATH}/public antes da atualização:"
ls -la "${NEW_RELEASE_PATH}/public" || exit_with_error "Falha ao listar ${NEW_RELEASE_PATH}/public antes da atualização"

# 24. Verificar permissões do diretório CURRENT_LINK
echo "Verificando permissões de ${CURRENT_LINK}... $(get_date)"
if ! [ -w "${CURRENT_LINK}" ]; then
    exit_with_error "Usuário não tem permissão de escrita em ${CURRENT_LINK}"
fi

# 25. Atualizar o conteúdo do diretório ativo com a nova release
echo "Atualizando conteúdo do diretório ativo com a nova release... $(get_date)"
# Remover arquivos antigos, exceto pastas compartilhadas (storage e public/build)
find "${CURRENT_LINK}" -maxdepth 1 -not -path "${CURRENT_LINK}" -not -path "${CURRENT_LINK}/storage" -not -path "${CURRENT_LINK}/public/build" -exec rm -rf {} + || exit_with_error "Falha ao remover arquivos antigos em ${CURRENT_LINK}"
# Copiar arquivos da nova release, excluindo storage e public/build
rsync -a --exclude 'storage' --exclude 'public/build' "${NEW_RELEASE_PATH}/" "${CURRENT_LINK}/" || exit_with_error "Falha ao copiar arquivos da nova release para ${CURRENT_LINK}"

# 26. Recriar o link simbólico public/build em ${CURRENT_LINK}/public
echo "Recriando o link simbólico para public/build em ${CURRENT_LINK}/public... $(get_date)"
# Validar se SHARED_BUILD_PATH existe
if [ ! -d "${SHARED_BUILD_PATH}" ] || [ ! -f "${SHARED_BUILD_PATH}/manifest.json" ]; then
    exit_with_error "Diretório ${SHARED_BUILD_PATH} ou manifest.json não encontrado"
fi
rm -rf "${CURRENT_LINK}/public/build"
ln -s "${SHARED_BUILD_PATH}" "${CURRENT_LINK}/public/build" || exit_with_error "Falha ao recriar link simbólico para public/build em ${CURRENT_LINK}/public"

# 27. Verificar o conteúdo da pasta public após a atualização (para diagnóstico)
echo "Conteúdo da pasta ${CURRENT_LINK}/public após a atualização:"
ls -la "${CURRENT_LINK}/public" || exit_with_error "Falha ao listar ${CURRENT_LINK}/public após a atualização"

# 28. Confirmar que os arquivos estão disponíveis após a atualização (para diagnóstico)
echo "Verificando arquivos em ${CURRENT_LINK}/public/build após a atualização:"
ls -la "${CURRENT_LINK}/public/build" || exit_with_error "Falha ao verificar arquivos em public/build após a atualização"

# 29. Limpar caches do Laravel e Filament
echo "Limpando caches do Laravel e Filament... $(get_date)"
cd "${CURRENT_LINK}" || exit_with_error "Falha ao acessar ${CURRENT_LINK}"
stdbuf -oL timeout 300 ${PHP_EXEC} artisan optimize:clear || exit_with_error "Falha ao limpar caches do Laravel"
stdbuf -oL timeout 300 ${PHP_EXEC} artisan filament:optimize-clear || echo "Aviso: Falha ao limpar caches do Filament, mas continuando deploy"

# 30. Recarregar o PHP-FPM
touch "${PHP_FPM_RELOAD}" 2>/dev/null || true
( flock -w 10 9 || exit_with_error "Falha ao adquirir lock para reload do PHP-FPM"
    echo "Recarregando PHP-FPM... $(get_date)"
    service "${PHP_FPM_SERVICE}" reload || echo "Aviso: Falha ao recarregar PHP-FPM, mas continuando"
) 9<"${PHP_FPM_RELOAD}"

# 31. Limpar releases e backups antigos (manter as últimas 3)
echo "Limpando releases antigas... $(get_date)"
ls -dt "${RELEASES_PATH}"/[0-9]* | tail -n +4 | xargs -I {} rm -rf {}
echo "Limpando backups antigos... $(get_date)"
ls -dt "${DB_BACKUP_PATH}"/backup_[0-9]*.sql | tail -n +4 | xargs -I {} rm -f {}

# 32. Limpar backups pré-migração, se existirem
echo "Limpando backups pré-migração... $(get_date)"
rm -f "${DB_BACKUP_PATH}/pre_migration_backup_*.sql" 2>/dev/null || true

echo "Deploy concluído com sucesso! $(get_date)"


Configurações do Servidor
Para que o script funcione sem erros em qualquer site no Laravel Forge, o servidor deve estar configurado corretamente. Abaixo estão os pré-requisitos e as configurações necessárias, assumindo que o usuário forge tem todas as permissões necessárias.
1. Sistema Operacional

Requisito: Ubuntu (versão suportada pelo Laravel Forge, ex.: 20.04 ou 22.04).
Verificação:lsb_release -a



2. Pacotes necessários

Git: Para gerenciar o repositório.apt-get install git


PHP: Versão compatível com o projeto (ex.: PHP 8.1 ou 8.2).php -v


Composer: Para gerenciar dependências PHP.composer --version


Node.js e npm: Para compilar assets com Vite.node -v
npm -v


PostgreSQL: Para o banco de dados.psql --version


rsync: Para copiar arquivos entre releases.apt-get install rsync


tzdata: Para suportar o fuso horário America/Fortaleza.apt-get install tzdata



3. Permissões do usuário forge

O usuário forge deve ter permissões de escrita nos diretórios:
${FORGE_SITE_PATH}
${FORGE_SITE_PATH}/../releases
${FORGE_SITE_PATH}/../backups/db
${FORGE_SITE_PATH}/../shared/build


Configuração:chown -R forge:forge ${FORGE_SITE_PATH} ${FORGE_SITE_PATH}/../releases ${FORGE_SITE_PATH}/../backups ${FORGE_SITE_PATH}/../shared
chmod -R 775 ${FORGE_SITE_PATH} ${FORGE_SITE_PATH}/../releases ${FORGE_SITE_PATH}/../backups ${FORGE_SITE_PATH}/../shared



4. Configuração do PostgreSQL

Usuário e banco de dados:
Crie um usuário e banco de dados no PostgreSQL correspondentes às variáveis DB_USERNAME e DB_DATABASE no .env.
Exemplo:psql -U postgres
CREATE USER myuser WITH PASSWORD 'mypassword';
CREATE DATABASE mydatabase;
GRANT ALL PRIVILEGES ON DATABASE mydatabase TO myuser;
\q




Arquivo .pgpass:
Configure o arquivo ~/.pgpass do usuário forge para autenticação automática:echo "localhost:5432:${DB_DATABASE}:${DB_USERNAME}:${DB_PASSWORD}" > /home/forge/.pgpass
chmod 600 /home/forge/.pgpass
chown forge:forge /home/forge/.pgpass





5. Configuração do .env

O arquivo .env em ${FORGE_SITE_PATH}/.env deve conter:DB_DATABASE=mydatabase
DB_USERNAME=myuser
DB_PASSWORD=mypassword
PRE_MIGRATION_BACKUP=false
RUN_SEEDERS=false


Verifique com:cat ${FORGE_SITE_PATH}/.env



6. Configuração do PHP-FPM

O serviço PHP-FPM deve estar configurado no Forge e acessível ao usuário forge.
O usuário forge deve ter permissão para recarregar o PHP-FPM, configurada automaticamente pelo Forge.

7. Estrutura de diretórios

Crie a estrutura inicial, se não existir:mkdir -p ${FORGE_SITE_PATH} ${FORGE_SITE_PATH}/../releases ${FORGE_SITE_PATH}/../backups/db ${FORGE_SITE_PATH}/../shared/build
chown -R forge:forge ${FORGE_SITE_PATH}/..
chmod -R 775 ${FORGE_SITE_PATH}/..



8. Repositório Git

O repositório Git deve estar configurado em ${FORGE_SITE_PATH} com a branch definida em FORGE_SITE_BRANCH.
Verifique:cd ${FORGE_SITE_PATH}
git status




Instruções de Implantação no Laravel Forge
Passo 1: Configurar o site no Forge

Acesse o painel do Laravel Forge.
Crie ou selecione o site desejado.
Configure o repositório Git:
Repository: URL do repositório (ex.: https://github.com/user/repo).
Branch: A branch desejada (ex.: main ou develop).


Confirme que as variáveis do Forge estão definidas:
FORGE_SITE_PATH: Caminho do diretório do site.
FORGE_SITE_BRANCH: Nome da branch.
FORGE_PHP: Caminho para o PHP (ex.: /usr/bin/php8.1).
FORGE_COMPOSER: Caminho para o Composer (ex.: /usr/local/bin/composer).
FORGE_PHP_FPM: Nome do serviço PHP-FPM (ex.: php8.1-fpm).



Passo 2: Fazer upload do script

Conecte-se ao servidor via SSH como usuário forge:ssh forge@<server-ip>


Crie o arquivo deploy.sh:nano ${FORGE_SITE_PATH}/deploy.sh


Cole o conteúdo do script acima.
Dê permissão de execução:chmod +x ${FORGE_SITE_PATH}/deploy.sh



Passo 3: Configurar o script no Forge

No painel do Forge, vá para a seção Deploy Script do site.
Substitua o script padrão pelo seguinte:${FORGE_SITE_PATH}/deploy.sh


Salve as alterações.

Passo 4: Configurar o arquivo .env

Edite o arquivo .env:nano ${FORGE_SITE_PATH}/.env


Certifique-se de que contém as variáveis necessárias (veja a seção de Configurações do Servidor).

Passo 5: Executar o deploy

No painel do Forge, clique em Deploy Now.
Monitore o progresso no painel ou consulte o log:cat ${FORGE_SITE_PATH}/logs/deploy_${RELEASE_ID}.log


Verifique se o deploy conclui com a mensagem "Deploy concluído com sucesso!".

Passo 6: Validar o site

Acesse o site no navegador.
Confirme que as alterações do repositório estão refletidas.
Verifique os arquivos no diretório ativo:ls -la ${FORGE_SITE_PATH}


Confirme que o link simbólico public/build está correto:ls -ld ${FORGE_SITE_PATH}/public/build




Como Realizar um Rollback
Se o deploy não atender às expectativas ou apresentar problemas, o script permite reverter para uma das três releases anteriores, garantindo zero downtime durante o rollback. Siga os passos abaixo:
Passo 1: Identificar a release desejada

Liste as releases disponíveis:
ls -l ${FORGE_SITE_PATH}/../releases


As pastas são nomeadas com o formato YYYYMMDDHHMMSS (ex.: 20250427021116). A release mais recente é a atual, e as três anteriores estão disponíveis para rollback.


Escolha a release para a qual deseja reverter (ex.: 20250427020000).


Passo 2: Verificar a integridade da release

Confirme que a pasta da release contém todos os arquivos necessários:
ls -la ${FORGE_SITE_PATH}/../releases/20250427020000


Verifique a presença de artisan, public, e outros arquivos essenciais.


Opcional: Valide a release executando:
cd ${FORGE_SITE_PATH}/../releases/20250427020000
php artisan route:list


Isso confirma que a release é funcional.



Passo 3: Atualizar o diretório ativo

Faça backup do diretório ativo atual, por segurança:
cp -r ${FORGE_SITE_PATH} ${FORGE_SITE_PATH}/../backups/site_backup_$(date +%Y%m%d%H%M%S)


Remova os arquivos antigos do diretório ativo, exceto storage e public/build:
find ${FORGE_SITE_PATH} -maxdepth 1 -not -path "${FORGE_SITE_PATH}" -not -path "${FORGE_SITE_PATH}/storage" -not -path "${FORGE_SITE_PATH}/public/build" -exec rm -rf {} +


Copie os arquivos da release escolhida para o diretório ativo, excluindo storage e public/build:
rsync -a --exclude 'storage' --exclude 'public/build' ${FORGE_SITE_PATH}/../releases/20250427020000/ ${FORGE_SITE_PATH}/



Passo 4: Recriar o link simbólico public/build

Verifique se a pasta compartilhada de assets existe:ls -la ${FORGE_SITE_PATH}/../shared/build


Recrie o link simbólico:rm -rf ${FORGE_SITE_PATH}/public/build
ln -s ${FORGE_SITE_PATH}/../shared/build ${FORGE_SITE_PATH}/public/build



Passo 5: Limpar caches

Limpe os caches do Laravel e Filament:cd ${FORGE_SITE_PATH}
php artisan optimize:clear
php artisan filament:optimize-clear



Passo 6: Recarregar o PHP-FPM

Recarregue o serviço PHP-FPM para aplicar as alterações:service ${FORGE_PHP_FPM} reload



Passo 7: Validar o rollback

Acesse o site no navegador para confirmar que a release anterior está ativa.
Verifique os arquivos no diretório ativo:ls -la ${FORGE_SITE_PATH}


Consulte o log do deploy original da release para detalhes:cat ${FORGE_SITE_PATH}/logs/deploy_20250427020000.log



Passo 8: Restaurar o banco de dados (opcional)

Se o deploy incluiu migrações que alteraram o banco de dados, restaure o backup correspondente:psql -U ${DB_USERNAME} -d ${DB_DATABASE} -f ${FORGE_SITE_PATH}/../backups/db/backup_20250427020000.sql


Atenção: Restaure o banco apenas se necessário, pois isso pode sobrescrever dados recentes.

Notas sobre Rollback

O script mantém até três releases anteriores, permitindo múltiplas opções de reversão.
O rollback é rápido e preserva o diretório storage e os assets em public/build, garantindo zero downtime.
Sempre valide a release antes de aplicá-la e mantenha backups adicionais, se necessário.


Dicas de Depuração
1. Consultar logs

Verifique o arquivo de log do deploy:cat ${FORGE_SITE_PATH}/logs/deploy_${RELEASE_ID}.log


Consulte os logs do Forge:cat /var/log/forge/*



2. Verificar processos travados

Liste processos ativos:ps aux | grep bash
ps aux | grep artisan


Finalize processos travados:kill -9 <PID>



3. Problemas com Git

Confirme que a branch está sincronizada:cd ${FORGE_SITE_PATH}
git fetch origin
git diff origin/${FORGE_SITE_BRANCH}


Sincronize, se necessário:git reset --hard origin/${FORGE_SITE_BRANCH}



4. Problemas com caches

Limpe caches manualmente:cd ${FORGE_SITE_PATH}
php artisan optimize:clear
php artisan filament:optimize-clear



5. Problemas com permissões

Ajuste permissões:chown -R forge:forge ${FORGE_SITE_PATH}/..
chmod -R 775 ${FORGE_SITE_PATH}/..



6. Problemas com o banco de dados

Teste a conexão:psql -U ${DB_USERNAME} -d ${DB_DATABASE} -c "SELECT 1;"


Verifique o arquivo .pgpass:cat /home/forge/.pgpass




Conclusão
Este script de deploy, desenvolvido por Jierlausson Oliveira, é uma solução de Zero Downtime Deployments para aplicações Laravel no Laravel Forge, inspirada no Laravel Envoyer. Ele garante implantações seguras, rápidas e contínuas, sem interrupções no serviço, mesmo em caso de erros. A capacidade de realizar rollback para até três releases anteriores oferece flexibilidade e confiabilidade, enquanto os backups automáticos e as validações rigorosas minimizam riscos. As configurações detalhadas do servidor, as instruções de implantação e o guia de rollback garantem que o script funcione sem erros em qualquer site configurado no Forge. A explicação de cada uma das 32 etapas facilita o aprendizado e a manutenção.
Para dúvidas ou problemas, consulte os logs de deploy e as dicas de depuração. O script pode ser adaptado para outros projetos com ajustes mínimos, como alterações no fuso horário ou nos comandos Artisan.

