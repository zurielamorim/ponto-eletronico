# Sistema de Ponto Eletrônico em Laravel (Dockerizado)

Sistema de ponto eletrônico simplificado, desenvolvido em LARAVEL 7.0 e PHP 7.4.
Esta versão foi adaptada para rodar em containers **Docker**, facilitando a instalação e corrigindo problemas de compatibilidade com ambientes modernos.

## 🔧 Funcionalidades

* Inclusão, exclusão e modificação de Funcionário
* Ponto de entrada e saída
* Justificativa de falta
* Ajuste e Aprovação de ponto
* Relatório mensal com percentual de presença

---

## 🚀 Instalação com Docker (Recomendado)

Siga os passos abaixo para rodar o projeto em qualquer servidor com Docker e Docker Compose instalados.

### 1. Configuração Inicial

Clone o repositório e entre na pasta:
```bash
git clone [https://github.com/SEU_USUARIO/SEU_REPOSITORIO.git](https://github.com/SEU_USUARIO/SEU_REPOSITORIO.git)
cd ponto-eletronico


Crie o arquivo de configuração de ambiente:

Bash

cp .env.example .env
Edite o arquivo .env para configurar o banco de dados e a URL:

Bash

nano .env
Alterações necessárias no .env: Certifique-se de apontar o host do banco para db (nome do serviço no Docker) e ajustar o Timezone.

Ini, TOML

APP_URL=http://SEU_IP_OU_DOMINIO:8000
APP_TIMEZONE=America/Sao_Paulo

DB_CONNECTION=mysql
DB_HOST=db
DB_PORT=3306
DB_DATABASE=laravel
DB_USERNAME=laravel
DB_PASSWORD=root
2. Subir o Ambiente
Inicie os containers (Servidor Web + Banco de Dados):

Bash

docker compose up -d --build
3. Instalar Dependências e Configurar
Instale as bibliotecas do Laravel (dentro do container):

Bash

docker compose exec app composer install
Gere a chave da aplicação:

Bash

docker compose exec app php artisan key:generate
Dê permissão de escrita nas pastas de log e cache:

Bash

docker compose exec app chmod -R 777 storage bootstrap/cache
Crie as tabelas e o usuário administrador padrão no banco de dados:

Bash

docker compose exec app php artisan migrate
Acesse o sistema em: http://SEU_IP:8000/painel

🔐 Configurando o Usuário Administrador
O sistema vem com um usuário padrão, mas utiliza criptografia SHA1. Siga os passos abaixo para definir seu CPF e Senha de administrador.

Acesse o terminal interativo do Laravel (Tinker):

Bash

docker compose exec app php artisan tinker
Execute o comando abaixo para atualizar o usuário padrão. Substitua SEU_NOVO_CPF e SUA_NOVA_SENHA pelos dados desejados.

PHP

DB::table('usuario')->where('id', 1)->update([
    'cpf' => 'SEU_NOVO_CPF', 
    'senha' => hash('sha1', 'SUA_NOVA_SENHA')
]);
Exemplo de uso:

PHP

// Define CPF para 12312312312 e Senha para 123456789
DB::table('usuario')->where('id', 1)->update(['cpf' => '12312312312', 'senha' => hash('sha1', '123456789')]);
Digite exit para sair.

Agora você pode logar no painel com os novos dados.

🛠️ Comandos Úteis
Parar o servidor:

Bash

docker compose down
Ver logs de erro:

Bash

docker compose logs -f app
Limpar cache de configuração (caso edite o .env):

Bash

docker compose exec app php artisan config:clear
📄 Licença
Este projeto está sob a licença MIT.

