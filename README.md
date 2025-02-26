<div align="center">

#  🔍📊 Arrematei Monitor



![version](https://img.shields.io/badge/version-1.0.0-blue)
![rails](https://img.shields.io/badge/Rails-7.1.2-red)
![ruby](https://img.shields.io/badge/Ruby-3.2.2-red)
![license](https://img.shields.io/badge/license-MIT-green)
</div>

<div align="center">
  <img src="/public/monitor.png" alt="Arrematei Monitor Logo" />
  <h3>Uma solução de DevOps all-in-one para monitoramento do site</h3>
</div>

## 📋 Visão Geral

**Arrematei Monitor** é uma ferramenta de DevOps completa construída em Ruby on Rails para monitorar o site `arrematei.cloud`. Esta solução elimina a necessidade de utilizar múltiplas ferramentas de terceiros, integrando todas as funcionalidades em uma única aplicação.

### ✨ Funcionalidades Principais

- **🔍 Monitoramento em Tempo Real**
  - Tempo de resposta do site
  - Taxa de erros
  - Carga do servidor
  - Uso de memória

- **📊 Análise de Visitantes**
  - Visualizações de página
  - Visitantes únicos
  - Taxa de rejeição
  - Duração média da sessão

- **⚡ Testes de Carga e Stress**
  - Configuração de testes com usuários simultâneos
  - Duração personalizada
  - Resultados detalhados de desempenho

- **🔔 Sistema de Alertas**
  - Notificações automáticas
  - Categorização por severidade
  - Histórico de alertas

- **📈 Dashboard Centralizado**
  - Visualização unificada
  - Gráficos interativos
  - Filtros por período

## 🛠️ Guia de Implementação

### Pré-requisitos

- Ruby 3.2.2
- Rails 7.1.2
- PostgreSQL
- Redis (para Sidekiq)
- Node.js (para testes de carga com Artillery)

### Passos para Instalação

#### 1. Criar um novo projeto Rails

```bash
rails new arrematei_monitor --database=postgresql
cd arrematei_monitor
```

#### 2. Configurar o Gemfile

Substitua o conteúdo do Gemfile pelo código abaixo:

```ruby
source 'https://rubygems.org'
git_source(:github) { |repo| "https://github.com/#{repo}.git" }

ruby '3.2.2'

# Rails padrão
gem 'rails', '~> 7.1.2'
gem 'pg', '~> 1.1'
gem 'puma', '~> 6.0'
gem 'sprockets-rails'
gem 'importmap-rails'
gem 'turbo-rails'
gem 'stimulus-rails'
gem 'jbuilder'
gem 'bootsnap', require: false

# Monitoramento e métricas
gem 'prometheus-client'             # Coletor de métricas
gem 'skylight'                      # Monitoramento de performance de aplicação
gem 'scout_apm'                     # APM (Application Performance Monitoring)
gem 'newrelic_rpm'                  # New Relic para monitoramento em tempo real

# Teste de carga/stress
gem 'artillery-core'                # Para testes de carga
gem 'benchmark-ips'                 # Benchmarking
gem 'rack-attack'                   # Proteção contra ataques

# Visualização e Analytics
gem 'chartkick'                     # Para gráficos
gem 'groupdate'                     # Agrupar dados por data
gem 'ahoy_matey'                    # Analytics
gem 'blazer'                        # Ferramenta de BI e consulta

# Notificações e alertas
gem 'exception_notification'        # Notificações de erro
gem 'slack-notifier'                # Integração com Slack

# Background jobs
gem 'sidekiq'                       # Para processamento em background
gem 'sidekiq-scheduler'             # Agendamento de tarefas

# Autenticação
gem 'devise'                        # Autenticação de usuários

group :development, :test do
  gem 'debug', platforms: %i[ mri mingw x64_mingw ]
  gem 'rspec-rails'
  gem 'factory_bot_rails'
  gem 'faker'
  gem 'pry-rails'
end

group :development do
  gem 'web-console'
  gem 'rack-mini-profiler'
  gem 'spring'
  gem 'annotate'
  gem 'rubocop-rails'
end

group :test do
  gem 'capybara'
  gem 'selenium-webdriver'
  gem 'webmock'
  gem 'vcr'
end
```

E então instale as gems:

```bash
bundle install
```

#### 3. Configurar o Banco de Dados

Edite o arquivo `config/database.yml` com suas configurações locais:

```yaml
default: &default
  adapter: postgresql
  encoding: unicode
  pool: <%= ENV.fetch("RAILS_MAX_THREADS") { 5 } %>
  username: seu_usuario
  password: sua_senha
  host: localhost

development:
  <<: *default
  database: arrematei_monitor_development

test:
  <<: *default
  database: arrematei_monitor_test

production:
  <<: *default
  database: arrematei_monitor_production
  username: arrematei_monitor
  password: <%= ENV["ARREMATEI_MONITOR_DATABASE_PASSWORD"] %>
```

Em seguida, crie o banco de dados:

```bash
rails db:create
```

#### 4. Executar as Migrações e Criar os Modelos

Execute os seguintes comandos:

```bash
# Modelos principais
rails g model Site url:string name:string status:string last_checked_at:datetime
rails g model VisitorMetric page_views:integer unique_visitors:integer bounce_rate:decimal average_session_duration:decimal date:date site:references
rails g model PerformanceMetric response_time:decimal error_rate:decimal server_load:decimal memory_usage:decimal date:datetime site:references
rails g model LoadTest name:string duration:integer concurrent_users:integer status:string results:jsonb started_at:datetime completed_at:datetime site:references
rails g model Alert name:string description:text status:string severity:string resolved_at:datetime site:references

# Configurar Devise para autenticação
rails g devise:install
rails g devise User email:string admin:boolean

# Executar migrações
rails db:migrate
```

#### 5. Implementar os Controllers e Workers

##### Controllers

Crie os seguintes controllers:

**`app/controllers/dashboard_controller.rb`**
```ruby
class DashboardController < ApplicationController
  before_action :authenticate_user!
  
  def index
    @sites = Site.all
    @current_site = params[:site_id] ? Site.find(params[:site_id]) : Site.first
    
    if @current_site
      # Métricas de visitantes para o site atual
      @visitor_metrics = @current_site.visitor_metrics.group_by_day(:date, last: 30).sum(:page_views)
      @unique_visitors = @current_site.visitor_metrics.group_by_day(:date, last: 30).sum(:unique_visitors)
      
      # Métricas de performance para o site atual
      @response_times = @current_site.performance_metrics.group_by_hour(:date, last: 24).average(:response_time)
      @error_rates = @current_site.performance_metrics.group_by_hour(:date, last: 24).average(:error_rate)
      
      # Alertas recentes
      @recent_alerts = @current_site.alerts.where(resolved_at: nil).order(created_at: :desc).limit(5)
      
      # Testes de carga recentes
      @recent_load_tests = @current_site.load_tests.order(created_at: :desc).limit(5)
    end
  end
end
```

**`app/controllers/sites_controller.rb`**
```ruby
class SitesController < ApplicationController
  before_action :authenticate_user!
  before_action :set_site, only: [:show, :edit, :update, :destroy, :check]
  
  def index
    @sites = Site.all
  end
  
  def show
    @recent_performance = @site.performance_metrics.order(date: :desc).limit(10)
    @recent_visitor_data = @site.visitor_metrics.order(date: :desc).limit(10)
  end
  
  def new
    @site = Site.new
  end
  
  def create
    @site = Site.new(site_params)
    
    if @site.save
      redirect_to @site, notice: 'Site foi adicionado com sucesso.'
    else
      render :new
    end
  end
  
  def edit
  end
  
  def update
    if @site.update(site_params)
      redirect_to @site, notice: 'Site foi atualizado com sucesso.'
    else
      render :edit
    end
  end
  
  def destroy
    @site.destroy
    redirect_to sites_path, notice: 'Site foi removido com sucesso.'
  end
  
  def check
    SiteMonitorWorker.perform_async(@site.id)
    redirect_to @site, notice: 'Verificação do site iniciada.'
  end
  
  private
  
  def set_site
    @site = Site.find(params[:id])
  end
  
  def site_params
    params.require(:site).permit(:url, :name)
  end
end
```

**`app/controllers/load_tests_controller.rb`**
```ruby
class LoadTestsController < ApplicationController
  before_action :authenticate_user!
  before_action :set_site
  before_action :set_load_test, only: [:show, :destroy]
  
  def index
    @load_tests = @site.load_tests.order(created_at: :desc)
  end
  
  def show
    @test_results = @load_test.results
  end
  
  def new
    @load_test = @site.load_tests.new
  end
  
  def create
    @load_test = @site.load_tests.new(load_test_params)
    @load_test.status = 'pending'
    
    if @load_test.save
      LoadTestWorker.perform_async(@load_test.id)
      redirect_to site_load_test_path(@site, @load_test), notice: 'Teste de carga iniciado.'
    else
      render :new
    end
  end
  
  def destroy
    @load_test.destroy
    redirect_to site_load_tests_path(@site), notice: 'Teste de carga removido.'
  end
  
  private
  
  def set_site
    @site = Site.find(params[:site_id])
  end
  
  def set_load_test
    @load_test = @site.load_tests.find(params[:id])
  end
  
  def load_test_params
    params.require(:load_test).permit(:name, :duration, :concurrent_users)
  end
end
```

##### Workers

Crie os seguintes workers:

**`app/workers/site_monitor_worker.rb`**
```ruby
class SiteMonitorWorker
  include Sidekiq::Worker
  
  def perform(site_id)
    site = Site.find(site_id)
    site.check_status
  end
end
```

**`app/workers/load_test_worker.rb`**
```ruby
class LoadTestWorker
  include Sidekiq::Worker
  
  def perform(load_test_id)
    load_test = LoadTest.find(load_test_id)
    load_test.run_test
  end
end
```

**`app/workers/site_monitor_schedule_worker.rb`**
```ruby
class SiteMonitorScheduleWorker
  include Sidekiq::Worker
  
  def perform
    Site.all.each do |site|
      SiteMonitorWorker.perform_async(site.id)
    end
  end
end
```

**`app/workers/collect_visitor_metrics_worker.rb`**
```ruby
class CollectVisitorMetricsWorker
  include Sidekiq::Worker
  
  def perform
    Site.all.each do |site|
      # Em produção, você integraria com Google Analytics ou outra fonte de dados
      # Aqui estamos simulando dados
      
      site.visitor_metrics.create!(
        page_views: rand(100..1000),
        unique_visitors: rand(50..500),
        bounce_rate: rand(10.0..70.0),
        average_session_duration: rand(30.0..300.0),
        date: Date.today - 1.day
      )
    end
  end
end
```

#### 6. Configurar o Sidekiq para Agendamento de Tarefas

Crie o arquivo `config/sidekiq.yml`:

```yaml
:schedule:
  site_monitor:
    every: '5m'
    class: SiteMonitorScheduleWorker
  collect_visitor_metrics:
    cron: '0 0 * * *'  # Diariamente à meia-noite
    class: CollectVisitorMetricsWorker
```

Adicione o Sidekiq à inicialização do Rails editando `config/routes.rb`:

```ruby
require 'sidekiq/web'
require 'sidekiq-scheduler/web'

Rails.application.routes.draw do
  devise_for :users
  
  authenticate :user, lambda { |u| u.admin? } do
    mount Sidekiq::Web => '/sidekiq'
  end
  
  resources :sites do
    member do
      post :check
    end
    
    resources :load_tests
    resources :alerts
  end
  
  root 'dashboard#index'
  get 'dashboard', to: 'dashboard#index'
end
```

### 7. Criar as Views

Crie as views necessárias, começando pelo dashboard:

**`app/views/dashboard/index.html.erb`**
(Usar o código fornecido anteriormente)

### 8. Adicionar Estilos

Crie o arquivo `app/assets/stylesheets/dashboard.css` com o código fornecido anteriormente.

### 9. Iniciar os Serviços

Inicie o Redis (necessário para o Sidekiq):

```bash
redis-server
```

Em outro terminal, inicie o Sidekiq:

```bash
bundle exec sidekiq
```

Em outro terminal, inicie o servidor Rails:

```bash
rails server
```

## 🔧 Configuração Avançada

### Integrações

- **Google Analytics**: Para dados reais de visitantes
- **New Relic**: Para monitoramento de performance avançado
- **Slack**: Para notificações em tempo real

### Deployment

Recomendamos usar:
- **Heroku** para uma implementação rápida
- **AWS** ou **DigitalOcean** para um ambiente de produção completo

```bash
# Exemplo de deploy para Heroku
heroku create arrematei-monitor
git push heroku main
heroku run rails db:migrate
heroku addons:create heroku-redis:hobby-dev
heroku addons:create heroku-postgresql:hobby-dev
```

## 📊 Dashboard

![Dashboard Preview](/public/dashboard.png)

O dashboard oferece uma visão completa do seu site, incluindo:

- **Métricas em tempo real**
- **Histórico de performance**
- **Alertas ativos**
- **Resultados de testes de carga**

## 🚨 Sistema de Alertas

Os alertas são categorizados em:

| Severidade | Condição | Ação |
|------------|----------|------|
| **Baixa** | Tempo de resposta > 1s | Monitoramento |
| **Média** | Tempo de resposta > 2s | Notificação |
| **Alta** | Tempo de resposta > 5s | Notificação + SMS |
| **Crítica** | Site offline | Notificação + SMS + Ligação |

## 💡 Dicas de Uso

- Configure verificações a cada 5 minutos em produção
- Mantenha alertas de severidade crítica apenas para falhas reais
- Execute testes de carga em horários de baixo tráfego
- Use o registro de logs para diagnóstico de problemas

## 📝 Licença

Este projeto está licenciado sob a [MIT License](LICENSE).

## 👥 Contribuidores

- Desenvolvido para arrematei.cloud
- Mantido por [Michael Bullet]

---

<div align="center">
  <p> ❤️ arrematei.cloud</p>
</div>
