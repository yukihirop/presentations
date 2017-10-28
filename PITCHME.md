# light_serviceについて

---

## light_serviceとは
[Railsコードを改善する7つの素敵なGem（翻訳）](https://techracho.bpsinc.jp/hachi8833/2017_10_25/47160)
で紹介されていた[interactor](https://github.com/collectiveidea/interactor)というgemの基本になったgem
のことである。

---

## interactor

interactorをrailsのAPplicationControllerで使用すると、以下のような
ユーザー認証処理を

```ruby
class SessionsController < ApplicationController
  def create
    if user = User.authenticate(session_params[:email], session_params[:password])
      session[:user_token] = user.secret_token
      redirect_to user
    else
      flash.now[:message] = "Please try again."
      render :new
    end
  end

  private

  def session_params
    params.require(:session).permit(:email, :password)
  end
end
```

+++

こんなふうに見通しのいいように書き換えることができるのだが
一体どのようなアーキテクチャーがあって実現できているのかを
見ていきたい。

```ruby
class SessionsController < ApplicationController
  def create
    result = AuthenticateUser.call(session_params)

    if result.success?
      session[:user_token] = result.token
      redirect_to result.user
    else
      flash.now[:message] = t(result.message)
      render :new
    end
  end

  private

  def session_params
    params.require(:session).permit(:email, :password)
  end
end
```
