# Обзор

Это стартовый шаблон для написания Android приложений с использованием **Чистой архитектуры**.
Вы можете скачать его, модифицировать его и начать строить свои приложения на его основе.
Большая часть кода стартового шаблона для создания вашего первого вида, презентера и интерактора уже написана 
 и вам лишь нужно реализовать свою собственную логику. Я написал [подробную инструкцию]  о том, как
 писать приложения, использя этот паттерн, но данный README содержит итоговую выжимку.

Это стартовое приложение поддерживает **API 15 и выше**.

Данный шаблон использует **обучную Java** вместо RxJava и не использует Dagger.
Хотя я рекомендую Dagger, причина для этого в том, что не хотелось дополнительно усложнять, т.к.
архитктура сама по себе вероятно достаточно сложна для понимания.
Если вы предпочитаете RxJava и Dagger, то можете взглянуть на прекрасный проект под названием [Android-CleanArchitecture] 
, который вдохновил меня для создания этого.

Чтобы увидеть пример приложения, использующего Чистую Архитектуру, можете посмотреть [здесь].

## Библиотеки, которые использованы

 - [Android Support Library] для обратной совместимости.
 - [Timber] для логирования.
 - [Butterknife] для инъекций в виды.
 - [Retrofit] для сетевого взаимодействия.
 - [JUnit] и [Mockito] для тестирования.
 - [Findbugs] для поиск а багов, *а вы как думали!*.

# Что нужно поменять

Вы захотите сделать некоторые незначительные изменения при использовании шаблона:

- Переименовать базовый пакет `com.kodelabs.boilerplate` на то имя, которое вы предпочитаете. *[Как это сделать]*
- Изменить `applicationId` в вашем файле `app/build.gradle` на базовый пакет приложения, который вы выбрали шагом выше.
- Изменить `package` в теге манифеста в файле AndroidManifest.xml

***Если вы создали и/или опубликовали приложение с использованием данного шаблона, дайте мне знать и я упомяну его в списке здесь.***

# Начинаем с написания нового use case

Use case это лишь некоторая изолированная функциональность приложения.
Use case может быть инициирован пользователем (например по нажатию пользователя), а может и не быть. 
Напрммер, use case мог бы быть следующий: *"Получить все данные из базы данных и отобразить их в UI при старте приложения"*.

В этом примере наш use case будет: ***"Поприветствовать пользователя сообщением, которое хранится в базе данных, когда приложение стартует."*** 
Этот пример может быть опробован в ветке `example`. Данный пример покажет как написать следующие три пакета, 
необходимые для того, чтобы сценарий заработал:

- **презентационные** слои 
- слои **хранилища**
- слои **предметной области**

Первые два относятся к внешним слоям, в то время как последний - к внутреннему слою ядра. 
Пакет **presentation** отвечает за все, что связано с отображением на экране  — он включает целый MVP стек (это означает, что он включает пакеты и UI и Презентера, хотя они относятся к различным слоям).


## **Создание нового интерактора(внутренний слой / слой ядра)**

В реальности вы могли бы начать с любого слоя архитетуры, но я рекомендую вам начать с бизнес-логики приложения.
Вы можете написать ее, оттестировать ее и убедиться, что она работает без создания какого-либо экрана активности.

Итак, начнем с создания интерактора. Интерактор - это место, где обитает главная логика сценария use-case. 
**Все интеракторы запускаются в фоновом треде, поэтому они не должны оказывать влияние на производительность UI.**
Давайте создадим новый интерактор с теплым именем `WelcomingInteractor`.

```java
public interface WelcomingInteractor extends Interactor {

    interface Callback {

        void onMessageRetrieved(String message);

        void onRetrievalFailed(String error);
    }
}

```

Интерфейс `Callback` ответственнен за общение с UI в главном потоке, мы помещаем все это в интерфейс Interactor-а, поэтому нам нет необходимости его именовать чем-то вроде 
***WelcomingInteractorCallback*** — чтобы отличать егоот других колбэков.
 Теперь давайте имплементируем нашу логику получения сообщения. Скажем, у нас есть некий `MessageRepository`, который может дать нам наше сообщение для приветствия.

```java
public interface MessageRepository {
    String getWelcomeMessage();
}
```
Теперь нам нужно реализовать наш интерфейс Interactor с собственной бизнес-логикой.
**Важно, чтобы реализация наследовала от класса `AbstractInteractor`, который берет на себя запуск его в фоновом потоке.**

```java
public class WelcomingInteractorImpl extends AbstractInteractor implements WelcomingInteractor {

...

  private void notifyError() {
      mMainThread.post(new Runnable() {
          @Override
          public void run() {
              mCallback.onRetrievalFailed("Nothing to welcome you with :(");
          }
      });
  }

  private void postMessage(final String msg) {
      mMainThread.post(new Runnable() {
          @Override
          public void run() {
              mCallback.onMessageRetrieved(msg);
          }
      });
  }

  @Override
  public void run() {

      // получаем сообщение
      final String message = mMessageRepository.getWelcomeMessage();

      // проверяем, провалились ли мы при получении нашего сообщения
      if (message == null || message.length() == 0) {

          // уведомляем о провале пользовательский интерфейс в главном потоке
          notifyError();

          return;
      }

      // мы получили наше сообщение, уведомляем UI главного потока
      postMessage(message);
  }
}
```
Здесь лишь попытка полчить сообщение и отправить сообщение или ошибок в UI для отображения. Мы уведомляем UI используя наш `Callback` который на самом деле будет нашим `Презентером`. **Это крест нашей бизнес-логики. Последнее, что нам нужно - это зависимость от фреймворка.**

Давайте взглянем на то, какие зависимости имеет этот Интерактор:

```java
import com.kodelabs.boilerplate.domain.executor.Executor;
import com.kodelabs.boilerplate.domain.executor.MainThread;
import com.kodelabs.boilerplate.domain.interactors.WelcomingInteractor;
import com.kodelabs.boilerplate.domain.interactors.base.AbstractInteractor;
import com.kodelabs.boilerplate.domain.repository.MessageRepository;
```

Вы можете видеть, что нет **ни единого упоминания об Android коде**. Это **главное преимущество** данного подхода. 
Кроме того, мы не беспокоимся о специфике UI базы данных, мы просто вызываем методы интерфейса, которые где-то во внешнем слое будут реализованы.

## **Тестирование нашего интерактора**

Мы теперь можем запустить и протестировать наш Интерактор без запуска эмулятора. Давайте напишем простой **JUnit** тест, чтобы убедиться, что он работает:

```java
@Test
public void testWelcomeMessageFound() throws Exception {

    String msg = "Welcome, friend!";

    when(mMessageRepository.getWelcomeMessage())
            .thenReturn(msg);

    WelcomingInteractorImpl interactor = new WelcomingInteractorImpl(
      mExecutor,
      mMainThread,
      mMockedCallback,
      mMessageRepository
    );
    interactor.run();

    Mockito.verify(mMessageRepository).getWelcomeMessage();
    Mockito.verifyNoMoreInteractions(mMessageRepository);
    Mockito.verify(mMockedCallback).onMessageRetrieved(msg);
}
```

Еще раз, данный код Интерактора даже не догадывается, что он будет жить внутри Андроид приложения.

## **Создание презентационного слоя**

Презентационный слой - это **внешний слой** в Чистой Архитектуре. Он состоит из фреймворко-зависимого кода для отображения UI пользователю.
Мы будем использовать класс `MainActivity` для отображения привественного сообщения пользователю, когда приложение запускается.

Давайте начнем с написания интерфейса нашего `Презентера` и `Вида`. Единственная вещь, которая нужна нашему `виду` это отобразить сообщение:

```java
public interface MainPresenter extends BasePresenter {

    interface View extends BaseView {
        void displayWelcomeMessage(String msg);
    }
}
```

Как и где мы стартуем Интерактор, когда приложение стартует? Все, что не связно жестко с видом, должно идти в класс `Презентера`. 
Это помогает дочись `разделение ответственности` и предохраняет класссы `Activity` от перегрузки. 
Он включает весь код, работающий с Интеракторами.

В нашем классе `MainActivity` мы перекрываем методе `onResume`:

```java
@Override
protected void onResume() {
    super.onResume();

    // давайте начнем получение приветственного сообщения, когда стартует приложение
    mPresenter.resume();
}
```

Все объекты `Презентеры` реализуют метод `resume()` когда наследуют от класса `BasePresenter`. 
Мы стартуем интерактор внутри класса `MainPresenter` в методе `resume()`:

```java
@Override
public void resume() {

    mView.showProgress();

    // инициализируем интерактор
    WelcomingInteractor interactor = new WelcomingInteractorImpl(
            mExecutor,
            mMainThread,
            this,
            mMessageRepository
    );

    // запускаем интерактор
    interactor.execute();
}
```

Метод `execute()` лишь запустит метод `run()` класса `WelcomingInteractorImpl` в фоновом потоке. Метод `run()` можем увидеть в разделе  ***Создание нового интерактора***.

Вы можете заметить, что Интерактор поведет схожим образом как и класс `AsyncTask`. 
Вы можете спросить, почему мы просто не использовали `AsyncTask`?
Т.к. это **Android специфичный код** и вам потребуется эмулятор для запуска его и тестирования его.

Мы предоставляем различные вещи в Интерактор:

- Объект `ThreadExecutor`, который ответственен за выполнение интеракторов в фоновом потоке. Я обычно делаю его синглтоном. Этот класс относится к категории `domain`, предметной области и не обязан быть реализован во внешнем слое.
- Объект `MainThreadImpl` который ответственен за запуск Runnable в главном потоке из кода интерактора. Главные потоки доступн, с использованием фреймворко-зависимого кода и мы реализуем его во внешнем слое `threading`.
- Вы можете отметить, что мы предоставляем `this` Интерактору — `MainPresenter` это объект `Callback`, который Интерактор будет использовать для уведомления UI по событиям.
- Мы предоставляем экземпляр класса `WelcomeMessageRepository` который реализует интерфейс `MessageRepository`, который использует наш интерактор. `WelcomeMessageRepository` раскрывается позднее в разделе ***Создание слоя хранения***.

Согласно `this`, `MainPresenter` класса `MainActivity` действительно реализует интерфейс `Callback`:

```java
public class MainPresenterImpl extends AbstractPresenter implements MainPresenter,
        WelcomingInteractor.Callback {
```

И это тот способ, которым мы ловим сигналы о событиях от Интерактора. Это код из `MainPresenterImpl`:

```java
@Override
public void onMessageRetrieved(String message) {
    mView.hideProgress();
    mView.displayWelcomeMessage(message);
}

@Override
public void onRetrievalFailed(String error) {
    mView.hideProgress();
    onError(error);
}
```

`Вид` как следует из этиз снипетов, это `MainActivity` который реализует этот интерфейс:

```java
public class MainActivity extends AppCompatActivity implements MainPresenter.View {
```

Который затем ответственен за отображение этих сообщений, как показано здесь:

```java
@Override
public void displayWelcomeMessage(String msg) {
    mWelcomeTextView.setText(msg);
}
```

И это все, что касается презентационного слоя.

## **Создание слоя хранения**

Это то, где нашрепозиторий имплементируется. Весь БД-специфичный код должен идти сюда. 
Паттерн репозиторий лишь абстрагирует, откуда приходят данные. 
Наша главная бизнес-логика безразлична к источнику данных  — будь это база данных, сервер или текстовые файлы.

Для сложных данных вы можете использовать [ContentProvider] или ORM инструменты, такие как [DBFlow]. 
Если вам необходимо получить данные из веба, то [Retrofit] вам в помощь. 
Если вам нужноо простое хранилище ключ-значение, то вы можете использовать [SharedPreferences]. Вы должны использовать подходящий инструмент для решения задачи. 

Наша база данных - на самом деле не СУБД. Это будет очень простой класс с симуляцией некоей задержки:

```java
public class WelcomeMessageRepository implements MessageRepository {
    @Override
    public String getWelcomeMessage() {
        String msg = "Welcome, friend!"; // будем дружелюбны

        // Давайте просимулируем некоторый временной лаг от сети/БД
        try {
            Thread.sleep(2000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }

        return msg;
    }
}
```
Что касается нашего `WelcomingInteractor`, лаг возможен, по причине работы с реальной сетью или подобной причине. Интерактор на самом деле не беспокоится, что скрывается под `MessageRepository` пока он реализует данный интерфейс.  


# Лицензия

`MIT`

[подробную инструкцию]: <https://medium.com/p/a-detailed-guide-on-developing-android-apps-using-the-clean-architecture-pattern-d38d71e94029>
[здесь]: <https://github.com/dmilicic/android-clean-sample-app>
[Как это сделать]: <https://stackoverflow.com/questions/16804093/android-studio-rename-package>
[Butterknife]: <https://github.com/JakeWharton/butterknife>
[Timber]: <https://github.com/JakeWharton/timber>
[Android Support Library]: <https://developer.android.com/tools/support-library/index.html>
[JUnit]: <https://github.com/junit-team/junit/wiki/Download-and-Install>
[Mockito]: <http://site.mockito.org/>
[Retrofit]: <https://square.github.io/retrofit/>
[Findbugs]: <http://findbugs.sourceforge.net/>
[DBFlow]: <https://github.com/Raizlabs/DBFlow>
[SharedPreferences]: <http://developer.android.com/training/basics/data-storage/shared-preferences.html>
[ContentProvider]: <http://developer.android.com/guide/topics/providers/content-providers.html>

[Android-CleanArchitecture]: <https://github.com/android10/Android-CleanArchitecture>
