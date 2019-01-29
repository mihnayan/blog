# Заметки по Spring Security

---
Дата последнего изменения: 19.11.2018
Версия Spring Security: "4.2.3.RELEASE"
---

## Компоненты ядра

### SecurityContextHolder, SecurityContext и объекты аутентификации

Компоненты ядра находятся в jar-файле `spirng-seicurity-core`.

Основополагающий объект - `SecurityContextHolder`, который хранит контекст безопасности (security context) приложения и содержит информацию о лице (principal), которое использует приложение в текущий момент. Для хранения этой информации **по умолчанию** используется `ThreadLocal`. Но это можно изменить в настройках.

#### Получение информации о текущем пользователе

Информация хранится в `SecurityContextHolder`. Для предоставления этой информации используется объект `Authentication`. Для примера можно использовать следующий код

```Java
Object principal = SecurityContextHolder.getContext().getAuthentication().getPrincipal();

if (principal instanceof UserDetails) {
    String username = ((UserDetails) principal).getUsername();
} else {
    String username = principal.toString();
}
```
Объект, возвращаемый методом `getContext()` - это экземпляр интерфейса `SecurityContex`. Основные механизмы аутентификации в Spring Security возвращают экземпляр `UserDetails` в качестве принципала.


## Как происходит процесс аутентификации в web-приложении

Упрощённо процесс аутентификации в веб приложении выглядит следующим образом.

Аутентификация происходит через фильтры, которые устанавливаются автоматически и образуют цепочку. Порядок установлен следующий:

* `o.s.s.web.access.channel.ChannelProcessingFilter`
* `o.s.s.web.context.SecurityContextPersistenceFilter`
* `o.s.s.web.session.ConcurrentSessionFilter`
* `o.s.s.web.authentication.UsernamePasswordAuthenticationFilter`
* `o.s.s.web.servletapi.SecurityContextHolderAwareRequestFilter`
* `o.s.s.web.jaasapi.JaasApiIntegrationFilter`
* `o.s.s.web.authentication.rememberme.RememberMeAuthenticationFilter`
* `o.s.s.web.authentication.AnonymousAuthenticationFilter`. Проверяет, содержит ли `SecurityContextHolder` объект `Authentication`. Если нет, то создаёт его. В качестве реализации использует `o.s.s.authentication.AnonymousAuthenticationToken`.
* `o.s.s.web.access.ExceptionTranslationFilter`. Отвечает за обнаружение любых выбрасываемых исключений Spring Security.
* `o.s.s.web.access.intercept.FilterSecurityInterceptor`

В данном случае точкой отправления будет `UsernamePasswordAuthenticationFilter`, который наследуется от абстрактного класса-фильтра `o.s.s.web.authentication.AbstractAuthenticationProcessingFilter`. У `UsernamePasswordAuthenticationFilter`а нет собственной реализации метода `doFilter()`, поэтому выполняется метод родителя, в котором производится попытка получить объект аутентификации (реализующий интерфейс `o.s.s.core.Authentication`):

```Java
Authentication authResult;

try {
    authResult = attemptAuthentication(request, response);

...

```

`attemptAuthentication()` реализуется уже непосредственно `UsernamePasswordAuthenticationFilter`. Здесь из http-запроса получаются сведения об имени пользователя и пароле, создаётся `UsernamePasswordAuthenticationToken`, который имплементирует интерфейс `Authentication` и, наконец, в доступном менеджере аутентификации, реализующем интерфейс `o.s.s.authentication.AuthenticationManager`, вызывается метод `authenticate()`. Конкретно, часть метода `UsernamePasswordAuthenticationFilter#attemptAuthentication` выглядит следующим образом:

```Java
...

UsernamePasswordAuthenticationToken authRequest =
        new UsernamePasswordAuthenticationToken(username, password);
...

return this.getAuthenticationManager().authenticate(authRequest);
```

где вызов `getAuthenticationManager()` возвращает объект, реализующий интерфейс `o.s.s.authentication.AuthenticationManager`.

Менеджером аутентификации в Spring Security является класс `o.s.s.authentication.ProviderManager`, который содержит список сконфигурированных провайдеров аутентификации, реализующих интерфейс `o.s.s.authentication.AuthenticationProvider`. При работе метода `AuthenticationManager#authenticate()` менеджер аутентификации перебирает последовательно доступные провайдеры, вызывая в свою очередь в них одноимённый метод, ровно до тех пор, пока какой-либо из провайдеров не вернёт **не нулевой** объект, реализующий `Authentication`. Возврат `null` будет означать, что данный провайдер не может произвести аутентификацию. Если же ни один из провайдеров не смог вернуть объект `Authentication`, то выбрасывается исключение `ProviderNotFoundException`.

Провайдеры аутентификации настраиваются в xml-файле (в случае конфигурации на основе пространства имён) в элементе `<authentication-provider>`:

```xml
<authentication-manager>
    <authentication-provider/>
        <user-service>
            <user authorities="ROLE_USER" name="user" password="user1" />
        </user-service>
    </authentication-provider>
</authentication-manager>
```
Если провайдер аутентификации явно не указан, то используется `o.s.s.authentication.dao.DaoAuthenticationProvider`. Когда в конфигурации Spring Security в элементе настройки менеджера аутентификации `<authentication-provider>` указывается подэлемент `user-service`, то это автоматически запускает реализацию `o.s.s.provisioning.InMemoryUserDetailsManager` интерфейса `o.s.s.provisioning.UserDetailsManager`, который, в свою очередь, расширяет `o.s.s.core.userdetails.UserDetailsService`.

Как сказано в документации к Spring Security, `DaoAuthenticationProvider` - самый простой `AuthenticationProvider`, реализуемый в Spring Security, который также является одним из самых ранних, поддерживаемых во фреймворке. Он использует `UserDetailsService` в качестве DAO для извлечения пользователя из системы хранения информации о пользователях: СУБД либо in-memory хранилища.

`DaoAuthenticationProvider` наследуюется от `o.s.s.authentication.dao.AbstractUserDetailsAuthenticationProvider`, который непосредственно имплементирует интерфейс `AuthenticationProvider`. При работе метода `DaoAuthenticationProvider#authenticate()`, провайдер использует доступный `UserDetailsService` (то есть, тот, который был указан в конфигурации) для извлечения информации о пользователе в виде объекта, реализующего интерфейс `o.s.s.core.userdetails.UserDetails`.

Интерфейс `UserDetailsService` содержит всего один метод:

```Java
UserDetails loadUserByUsername(String username) throws UsernameNotFoundException
```
Имя пользователя передаётся то, которое было получено в объекте `Authentication`, который в свою очередь был передан в менеджер аутентификации в виде `UsernamePasswordAuthenticationToken`. Как видно из описания метода, если пользователь не найден, то генерируется исключение `UsernameNotFoundException`.

После получения сведений о пользователе  в виде `UserDetails`, провайдер `DaoAuthenticationProvider` проводит проверку пользователя на соответствие полученных учётных данных учётным данным из `UsernamePasswordAuthenticationToken`. Если проверка проходит, то аутентификация считается успешной, в ином случае генерируется `AuthenticationException`.

При успешной аутентификации данные пользователя из `UserDetails` копируются в объект `Authentication` и возвращаются менеджеру аутентификации. Таким образом, `AuthenticationManager` получает не нулевой объект `Authentication`, что и требовалось изначально.



## Разное


### Где хранится веб-форма аутентификации по умолчанию?

При конфигурации на основе пространства имён, если нет указаний на конкретную форму авторизации, то применяется форма по умолчанию. Эта форма "зашита" в фильтре `o.s.s.web.authentication.ui.DefaultLoginPageGeneratingFilter`, который будет автоматически добавлен в цепочку фильтров.
