cancancan + rolify + devise + activeadmin
=
cancan과 rolify, devise 조합은 자주 쓰이는 조합입니다.<br>
유저에게 role을 주어서 권한을 얻게 하는 것인데 이를 activeadmin <br>
즉 어드민 페이지 접속 권한으로 사용해보겠습니다.
## 1. 준비 사항 및 default activeadmin 사용 
* Gemfile 설치
```ruby
[...]
gem 'activeadmin', github: 'activeadmin'
gem 'cancancan'
gem 'rolify'
gem 'devise'
[...]
```
* bash
```bash
bundle install
rails g active_admin:install
rake db:migrate
rake db:seed
```
여기까지 일반적인 active_admin 설정 방법입니다. <br>
AdminUser라는 모델로 어드민 계정을 관리합니다.<br>
이를 cancan 과 rolify로 하나의 User 모델에서 role을 통해 관리해보겠습니다.

## 2. user 모델 설정(cancan + rolify 적용)
* bash - devise로 user 모델 생성
```bash
rails g devise user
rake db:migrate
```
* bash - cancan & Role add
```bash
rails g cancan:ability
rails g rolify Role User
rake db:migrate
```
## 3. 권한 설정 및 능력 부여
* app/models/role.rb
```ruby
[...]
belongs_to :resource,
         :polymorphic => true
        # ,
        # :optional => true
[...]
```
:optional -> true 주석 처리
* app/models/ability.rb - 기본적인 cancan 능력 설정
```ruby
[...]
if user.has_role? :admin
    can :manage, :all
else
    cannot :manage, ActiveAdmin::Page
end
[...]
```
user의 권한이 admin이면 모든 것을 허용하고 아니라면 active_admin의 접근을 불허해라
* app/controllers/application_controller.rb - 거부되었을때의 코드 작성
```ruby
[...]
# 이건 일반 cancan 거부 코드
rescue_from CanCan::AccessDenied do |exception|
    respond_to do |format|
      format.json { head :forbidden, content_type: 'text/html' }
      format.html { redirect_to main_app.root_url, notice: exception.message }
      format.js   { head :forbidden, content_type: 'text/html' }
    end
end
# 밑에 거는 active_admin 전용 거부 코드
def access_denied(exception)
    redirect_to users_path, notice: exception.message
end
[...]
```
## 4. active_admin에 적용
* config/initializers/active_admin.rb
```ruby
[...]
# config.authentication_method = :authenticate_admin_user!
config.authorization_adapter = ActiveAdmin::CanCanAdapter
config.cancan_ability_class = "Ability"
config.on_unauthorized_access = :access_denied
config.current_user_method = :current_user
config.logout_link_path = :destroy_user_session_path
config.logout_link_method = :delete
[...]
```
주석 처리하고 나머지 요소는 주석을 풀어주거나 변경해줍니다.
## 5. 실제 적용
* bash
```bash
rails g controller home index
```
* config/routes.rb
```ruby
[...]
root 'home#index'
[...]
```
* db/seeds.rb
```ruby
user=User.create(email: 'admin@example.com', password: 'password', password_confirmation: 'password') if Rails.env.development?
user.add_role :admin
```
전에 있던 코드를 변경해서 User 모델에서 admin@example.com 계정을 admin role로 추가해줍니다.
* bash 
```ruby
rake db:seed
```
* app/views/layout/application.html.erb
```html
[...]
<% if flash[:notice] %>
  <div class="notice"><%= flash[:notice] %></div>
<% end %>
[...]
```
이렇게 flash 메세지를 추가해줍니다.
* 주소/users/sign_in 에 접속하셔서 admin 계정으로 로그인하고 주소/admin 에 접속해봅니다. -> success
* 로그아웃 후 주소/users/sign_up에서 새 계정을 만들고 주소/admin에 접속해봅니다. -> fail.. flash 메세지
* 완성 이후 필요없어진 AdminUser 모델은 [삭제](http://patrickperey.com/2014/03/13/rails-remove-a-table/)해주고 [active_admin](https://github.com/activeadmin/activeadmin/wiki)에서도 텝을 지워주면 됩니다.
## Reference Post
>[cancancan](https://github.com/CanCanCommunity/cancancan)<br>
>[rolify](https://github.com/RolifyCommunity/rolify)<br>
>[devise](https://github.com/plataformatec/devise)<br>
>[devise+cancan+rolify](https://github.com/RolifyCommunity/rolify/wiki/Devise---CanCanCan---rolify-Tutorial)<br>
>[ActiveAdmin Using the CanCan Adapter](http://github-docs.activeadmin.info/13-authorization-adapter.html#using-the-cancan-adapter)<br>
>[rails remove a table](http://patrickperey.com/2014/03/13/rails-remove-a-table/)<br>
>[active_admin 괜찮은 블로그](http://wantknow.tistory.com/70)<br>
>[using flash message](https://agilewarrior.wordpress.com/2014/04/26/how-to-add-a-flash-message-to-your-rails-page/)