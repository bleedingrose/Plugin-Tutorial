Plugin-Tutorial
===============
перевод [руководства](http://www.redmine.org/projects/redmine/wiki/Plugin_Tutorial) по созданию плагинов для Redmine

##руководство по созданию плагинов Redmine

данное руководство расчитано на Redmine 2.x. Руководство по   созданию плагинов для Redmine 1.x
[здесь](http://www.redmine.org/projects/redmine/wiki/Plugin_Tutorial?version=66). Данное руководство предпологает что вы знакомы Ruby on Rails framework.

###Создание новых плагинов
Рекомендуется установить переменную окружения RAILS_ENV используя комманду ниже
<pre>
$ export RAILS_ENV="production"
</pre>
в Windows
<pre>
$ export RAILS_ENV=production
</pre>
Создать новый плагин вы можете используя генератор плагинов Redmine.
Генератор плагинов Redmine имеет следующий синтаксис

ruby script/rails generate redmine_plugin &lt;plugin_name&gt;.

Итак откройте командную строку и перейдите в каталог redmine,
затем выполните следуюшую команду:

<pre>
$ ruby script/rails generate redmine_plugin Polls
      create  plugins/polls/app
      create  plugins/polls/app/controllers
      create  plugins/polls/app/helpers
      create  plugins/polls/app/models
      create  plugins/polls/app/views
      create  plugins/polls/db/migrate
      create  plugins/polls/lib/tasks
      create  plugins/polls/assets/images
      create  plugins/polls/assets/javascripts
      create  plugins/polls/assets/stylesheets
      create  plugins/polls/config/locales
      create  plugins/polls/test
      create  plugins/polls/README.rdoc
      create  plugins/polls/init.rb
      create  plugins/polls/config/routes.rb
      create  plugins/polls/config/locales/en.yml
      create  plugins/polls/test/test_helper.rb
</pre>

Данная команда создала скелет вашего плагина в папке plugins/pools. Отредактируйте файл plugins/pools/init.rb и внесите туда необходимую информацию о плагине (название, Автор, описание, и версия)
<pre>
Redmine::Plugin.register :polls do
  name 'Polls plugin'
  author 'John Smith'
  description 'A plugin for managing polls'
  version '0.0.1'
end
</pre>
Перезапустите приложение и зайдите в браузере на страницу (http://localhost:3000/admin/plugins)
После входа под профилем админа вы сможете увидить новый плагин в списке плагинов
![новый плагин в списке плагинов](http://www.redmine.org/attachments/download/7656/plugins_list1.png)  
Примечание: любые изменения init.rb файл плагина требует перезапуска приложения, поскольку оно не перезагружает init.rb после каждого запроса на каждый запрос.

###Генерация модели

Сейчас плагин не может сохронять никакую информацию. Давайте создадим простую модель опроса(Poll) для нашего плагина.
Для создания модели используйте следующий синтаксис:
<pre>
ruby script/rails generate redmine_plugin_model &lt;plugin_name&gt &lt;model_name&gt; [field[:type][:index] field[:type][:index] ...]
</pre>
выполним команду в командной строке
<pre>
$ ruby script/rails generate redmine_plugin_model polls poll question:string yes:integer no:integer
      create  plugins/polls/app/models/poll.rb
      create  plugins/polls/test/unit/poll_test.rb
      create  plugins/polls/db/migrate/001_create_polls.rb
</pre>
Это действие создаст модель Poll и соответствующий файл миграции  file 001_create_polls.rb в plugins/polls/db/migrate:
<pre>
class CreatePolls &lt; ActiveRecord::Migration
  def change
    create_table :polls do |t|
      t.string :question
      t.integer :yes, :default => 0
      t.integer :no, :default => 0
    end
  end
end
</pre>
Вы можете настроить ваш файл миграции (те. значения по умолчанию...) затем перенести базу данных с помощью следующей команды(прогнать миграцию):
<pre>
$ rake redmine:plugins:migrate

Migrating polls (Polls plugin)...
==  CreatePolls: migrating ====================================================
-- create_table(:polls)
   -> 0.0410s
==  CreatePolls: migrated (0.0420s) ===========================================
</pre>
Обратите внимание, что каждый модуль имеет свой собственный набор миграций.

Давайте добавим несколько опросов в консоли, так что нам было с чем работать. Консоль, где вы можете в интерактивном режиме работы и изучить Redmine среды и является очень информативной, чтобы поиграть. Но сейчас нам просто необходимо создать два Poll объекта
<pre>
ruby script/rails console
&#91;rails 3&#93; rails console
>> Poll.create(:question => "Can you see this poll")
>> Poll.create(:question => "And can you see this other poll")
>> exit
</pre>
Отредактируйте plugins/polls/app/models/poll.rb добавте метод vote который будет вызыватся из нашего контролера:
<pre>
class Poll &lt; ActiveRecord::Base
  def vote(answer)
    increment(answer == 'yes' ? :yes : :no)
  end
end
</pre>

###Генерация контролера

Сейчас плагин ничего не делает. Поэтому давайте создадим контроллер для него.
Мы будем использовать генератор контролера для плагинов redmine. Со следующим синтаксисом:

<pre>
ruby script/rails generate redmine_plugin_controller &lt;plugin_name&gt; &lt;controller_name&gt; [&lt;actions&gt;]
</pre>

Выполните эту команду в консоле
<pre>
$ ruby script/rails generate redmine_plugin_controller Polls polls index vote
      create  plugins/polls/app/controllers/polls_controller.rb
      create  plugins/polls/app/helpers/polls_helper.rb
      create  plugins/polls/test/functional/polls_controller_test.rb
      create  plugins/polls/app/views/polls/index.html.erb
      create  plugins/polls/app/views/polls/vote.html.erb
</pre>
Это создаст контролер с двумя методами(actions)(#index и #vote).
Отредактируйте файл plugins/polls/app/controllers/polls_controller.rb и внесите в два этих метода код необходимый для их работы
<pre>
class PollsController < ApplicationController
  unloadable

  def index
    @polls = Poll.all
  end

  def vote
    poll = Poll.find(params[:id])
    poll.vote(params[:answer])
    if poll.save
      flash[:notice] = 'Vote saved.'
    end
    redirect_to :action => 'index'
  end
end
</pre>
Затем отредактируйте plugins/polls/app/views/polls/index.html.erb, который будет отображать существующие опросы:
<pre>
&lt;h2&gt;Polls&lt;/h2&gt;

&lt;% @polls.each do |poll| %&gt;
  &lt;p&gt;
  &lt;%= poll.question %&gt;?
  &lt;%= link_to 'Yes', { :action => 'vote', :id => poll[:id], :answer => 'yes' }, :method => :post %&gt; (&lt;%= poll.yes %&gt;) /
  &lt;%= link_to 'No', { :action => 'vote', :id => poll[:id], :answer => 'no' }, :method => :post %&gt; (&lt;%= poll.no %&gt;)
  &lt;/p&gt;
&lt;% end %&gt;
</pre>
Вы можете удалить плагины/polls/app/views/polls/vote.html.erb поскольку визуализация данного файла выполнятся не будет.

###Добавление Роутинга(маршруты) 
Redmine не поддерживает подстановки по умолчанию маршрут (':controller/:action/:id').
Плагины должны декларировать маршруты, по которым они нуждаются в файле config/routes.rb.
Теперь отредактируйте plugins/polls/config/routes.rb и добавте в него 2 маршрута для двух действий
<pre>
get 'polls', :to => 'polls#index'
post 'post/:id/vote', :to => 'polls#vote'
</pre>
вы можете найти больше информации о Роутинге в Rails сдесь [http://guides.rubyonrails.org/routing.html](http://guides.rubyonrails.org/routing.html)
Теперь перезапустите ваше приложение и зайдите в браузере на страницу [http://localhost:3000/polls.](http://localhost:3000/polls)
Вы должны увидеть 2 опроса и сможете проголосовать:
![Плагин для Redmine](http://www.redmine.org/attachments/download/853/pools1.png)

###интернационализация

Перевод файлы должны быть сохранены вconfig/locales, например. plugins/polls/config/locales/.

###Добавление пункта меню принадлежащего плагину
Наш контроллер работает нормально, но пользователи должны знать url, чтобы увидеть опросы. С помощью Redmine plugin API, вы можете расширить стандартные меню(добавить в них свой пункт меню). Поэтому давайте добавим новый элемент в меню приложения.

<pre>
Redmine::Plugin.register :redmine_polls do
  [...]

  menu :application_menu, :polls, { :controller => 'polls', :action => 'index' }, :caption => 'Polls'
end
</pre>

Синтаксис:

<pre>
  menu(menu_name, item_name, url, options={})
</pre>

Существует пять меню, которые можно расширить:

* :top_menu - в верхнем левом меню
* :account_menu - в верхнем правом рядом в ссылками войти зарегистрировать
* :application_menu - главное меню отображается, когда пользователь находится не внутри проекта
* :project_menu - главное меню отображается, когда пользователь находится внутри проекта
* :admin_menu - меню отображается на странице администрирования (можете вставить после установки, прежде чем Plugins)

Доступные варианты:

* :param - параметр ключ, используемый для проекта id (по умолчанию :id)
* :if - Proc, которая вызывается перед отображением элемента, элемент отображается, только если она возвращает true,
* :caption - меню заголовка, который может содержать:
	* локализованную строку Symbol (например :plugin_local_name)
	* строку
	* Proc, которые могут принять проект в качестве аргумента
* :before, :after - место где в меню должен быть вставлен элемент (например. :after => :activity)
* :first, :last - если установлено в true, то элемент будет встать в начало или в конец меню (например. :last => true)
* :html - hash с html параметрами, передаваемые link_to при отображении пункта меню

В нашем примере мы добавили элемент в меню приложения, которая не содержит элементов по умолчанию.
Перезапустите redmine и перейдите [http://localhost:3000](http://localhost:3000)
![http://www.redmine.org/attachments/download/854/application_menu.png](http://www.redmine.org/attachments/download/854/application_menu.png)

Теперь вы можете получить доступ к опросам, нажав на вкладке Polls с экрана приветствия.

###Расширение меню проекта

Теперь, давайте рассмотрим, случай когда необходимо чтобы опросы задавались на уровне проекта (даже если это не так, в нашем примере).Мы хотели бы добавить Вкладку Polls в меню проекта, а не в меню приложения
Откройте init.rb и замените строку, которую мы добавляли ранее ( menu :application_menu, :polls, { :controller => 'polls', :action => 'index' }, :caption => 'Polls')
на следующие две
<pre>
Redmine::Plugin.register :redmine_polls do
  [...]

  permission :polls, { :polls => [:index, :vote] }, :public => true
  menu :project_menu, :polls, { :controller => 'polls', :action => 'index' }, :caption => 'Polls', :after => :activity, :param => :project_id
end
</pre>

Вторая строка добавляет наши опросы вкладки в меню проэкта, сразу после вкладки activity. Первая строка необходима и заявляет, что наши два метода из PollsController являются публичными. Мы вернемся позже, чтобы объяснить это более подробно. Перезапустить приложение и перейдите к одному из ваших проектов:
![http://www.redmine.org/attachments/download/3773/project_menu.png](http://www.redmine.org/attachments/download/3773/project_menu.png)

Если вы щелкните на вкладке Polls(в 3-й позиции), то вы должны заметить, что меню проекта не отображается.
Чтобы сделать проект меню видимой, вы должны инициализировать в контроллере инстансную переменную проекта (@project)

Для этого отредактируйте PollsController следующим образом
<pre>
def index
  @project = Project.find(params[:project_id])
  @polls = Poll.find(:all) # @project.polls
end
</pre>

id проекта доступна в param[:project_id] 
Теперь вы должны увидеть меню проекта находясь на вкладке Polls
![http://www.redmine.org/attachments/download/3774/project_menu_pools.png](http://www.redmine.org/attachments/download/3774/project_menu_pools.png)

###Добавление новых прав

Сейчас любой желающий может проголосовать и посмотреть опросы. Давайте сделаем ее более гибкую настройку прав
Мы собираемся создать2 разрешения : одно для просмотра опросы и другая для голосования. Эти разрешения не публичные(опция :public => true удаляется).
отредаетируйте plugins/polls/init.rb и замените строчку с permission на две следующих
<pre>
  permission :view_polls, :polls => :index
  permission :vote_polls, :polls => :vote
</pre>
Перезапустите redmine и перейдите [http://localhost:3000/roles/permissions:](http://localhost:3000/roles/permissions)
![http://www.redmine.org/attachments/download/858/permissions1.png](http://www.redmine.org/attachments/download/858/permissions1.png)
Теперь вы настраивать эти два разрешения для существующих ролей.
Конечно, небходимо добавить код в PollsController который сделает эту зашиту фактической в соответствии привилегиями текущего пользователя.
Для этого нам всего лишь нужно добавить :authorize filter и сделать получение инстансной переменной @project возможной только после отработки данного фильтра
Вот как это будет выглядеть для метода #index.
<pre>
class PollsController < ApplicationController
  unloadable

  before_filter :find_project, :authorize, :only => :index

  [...]

  def index
    @polls = Poll.find(:all) # @project.polls
  end

  [...]

  private

  def find_project
    # @project variable must be set before calling the authorize filter
    @project = Project.find(params[:project_id])
  end
end
</pre>
Для метода vote данная проверка прав может быть сделана по аналогии
После этого поросмотр опросов и возможность голосовать будет доступна только админам и пользователям для ролей которых выставленны соответствующие разрешения

Если вы хотите ваши разрешения в на различных языках, вам нужно добавить необходимые текстовые метки в файле языка.
Просто создайте файл соответствующий нужному языку *.yml (например. en.yml для английского) в папке plugins/polls/config/locales
и внесите туда метки и перевод для них например как указано ниже 
<pre>
"en":
  permission_view_polls: View Polls
  permission_vote_polls: Vote Polls
</pre>
В этом примере создается файл именем en.yml, но для всех остальных языков можно создать файлы анологично.
Как вы можете видеть на примере выше, метки соответствуют символам :view_polls и :vote_polls с дополнительным permission_.

Перезапустите redmine и укажите необходимые права

###Создание модуля проекта

Теперь функции опроса доступны для всех ваших проектов.Но вы хотите иметь возможность включать ее только для некторых проектов
Итак, давайте создадим 'Polls' модуль проекта. Это делается путем оборачивания получение разряшений методом #project_module.

Отредактируите init.rb и измените декларацию permissions(прав)
<pre>
 project_module :polls do
    permission :view_polls, :polls => :index
    permission :vote_polls, :polls => :vote
  end
</pre>
Перезапустите redmine и перейдите в настройки проекта
Щелкнете не вкладке Modules. И вы сможете модуль Polls в списке модулей (выключенный по умолчанию)
![http://www.redmine.org/attachments/download/859/modules.png](http://www.redmine.org/attachments/download/859/modules.png)
Теперь вы можете включать и выключать опросы для различных проектов

###Улучшение внешнего вида проекта
####Добавление стилей
Давайте начнем с добавления стилей для нашего плагина.
Создайте файл с именем voting.css в каталоге plugins/polls/assets/stylesheets
<pre>
    a.vote { font-size: 120%; }
    a.vote.yes { color: green; }
    a.vote.no  { color: red; }
</pre>

При запуске приложения, файлы из файлопровода плагина (plugins/polls/assets)
будут скопированы в public/plugin_assets/polls/ это необходимо чтобы сделать их доступными для вашего web-сервера.
Поэтому любые изменения, таблицы стилей или javascripts в плагине требуют перезапуска Redmine.
Используемые в стилях классы необходимо обьявить в ссылках. для этого измените их в файле plugins/polls/app/views/polls/index.html.erb
следующим образом
<pre>
&lt;%= link_to 'Yes', {:action => 'vote', :id => poll[:id], :answer => 'yes' }, :method => :post, :class => 'vote yes' %&gt; (&lt;%= poll.yes %&gt;)
&lt;%= link_to 'No', {:action => 'vote', :id => poll[:id], :answer => 'no' }, :method => :post, :class => 'vote no' %&gt; (&lt;%= poll.no %&gt;)
</pre>
Затем добавьте следующие строки в конце index.html.erb так, что ваша Таблица стилей может попасть в head страницы по Redmine:
<pre>
&lt;% content_for :header_tags do %&gt;
    &lt;%= stylesheet_link_tag 'voting', :plugin => 'polls' %&gt;
&lt;% end %&gt;
</pre>
Обратите внимание, что :plugin => 'polls' параметр требуется при вызове stylesheet_link_tag хелпера.

Скрипты могут быть включены в плагин представлений с использованием javascript_include_tag хелпера таким же образом.
####Установка заголовка страницы
Вы можете задать title страницы внутри вашего представления с помощью html_title хелпера.
<pre>
  &lt;% html_title "Polls" %  &gt;
</pre>

###Использование хуков(hooks)

####Хуки в отображении
Хуки в отображениях в редмайне позволяют вставлять ваш контент в заранее определенные места 
в шаблонов. для примера посмотите например код представленный по следующей ссылке [http://www.redmine.org/projects/redmine/repository/entry/tags/2.0.0/app/views/projects/show.html.erb#L52](http://www.redmine.org/projects/redmine/repository/entry/tags/2.0.0/app/views/projects/show.html.erb#L52) вы увидете два хука: первый имеет имя :view_projects_show_left и предназначен для вставки контента в левую часть шаблона и второй с именем :view_projects_show_right для вставки контента в правую часть шаблона. 

Чтобы использовать один или несколько хуков в отображениях вы должны создать класс наследуемый от  Redmine::Hook::ViewListener и реализующий методы хуков которые вы желаете использовать. Теперь чтобы выводить контент для определенных хуков на вкладке "обзор" в проектах вам вам необходимом подключить ваш клас в файле init.rb и реализовать методы с именами хуков которые хотите использовать.

Для нашего плагина создайте файл plugins/polls/lib/polls_hook_listener.rb и добавте в него следующий код:
<pre>
class PollsHookListener < Redmine::Hook::ViewListener
  def view_projects_show_left(context = {})
    return content_tag("p", "Custom content added to the left")
  end

  def view_projects_show_right(context = {})
    return content_tag("p", "Custom content added to the right")
  end
end
</pre>

Также добавте строчку в конец файла plugins/polls/init.rb:
<pre>
require_dependency 'polls_hook_listener'
</pre>

Перезапустите Redmine и перейдите на вкладку "обзор" проекта. Где высможете увидить две наших строчки слева и справа. 

Вы также можете использовать хелпер render_on для отображения частичных шаблонов. В нашем плагине замените содержимое файла plugins/polls/lib/polls_hook_listener.rb следующим:
<pre>
class PollsHookListener < Redmine::Hook::ViewListener
  render_on :view_projects_show_left, :partial => "polls/project_overview" 
end
</pre>

Добавте частичный шаблон в наш плагин для этого создайте файл app/views/polls/_project_overview.html.erb. Добавте в него контент (Например "Это сообщение выведено с использованием хуков") который будет выведен в левой части вкладки "обзор" на странице проекта. Не забудте перезапустить Redmine.

####Хуки в контролерах
В работе

