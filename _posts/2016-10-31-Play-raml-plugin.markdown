---
layout: post
title: "RAML роутинг в play framework"
comments: true
categories: programming
---

![Play Logo](https://www.playframework.com/assets/images/logos/play_full_color.svg)

[Play framework](https://www.playframework.com/documentation/2.5.x/Home) очень гибкий инструмент, но информации о том, как изменить [формат route файла](https://www.playframework.com/documentation/2.5.x/ScalaRouting#The-routes-file-syntax) на просторах интернета мало. В данной статье я расскажу о том, как можно заменить стандартный язык описания маршрутов на основе route файла на описание в формате [RAML](http://raml.org/). Для этого нам придется создать [свой sbt плагин](https://github.com/bavadim/raml2play).

##Постановка задачи

Я с командой работаю над большой и сложной [системой онлайн банкинга](https://auth.rbo.raiffeisen.ru). В нашей команде особое внимание уделяется документированию интерфейсов и тестам. Однажды я задался вопросом, можно ли объединить тесты, документацию и кодогенерацию вместе. Оказалось, что до определенной степени это возможно. Обычно в документации содержится информация о rest-эндпоинтах, описание параметров и тела запроса, http-коды и описание ответов. При этом для всех вышеперечисленных элементов часто указываются примеры. Этой информации вполне достаточно для того, чтобы протестировать работоспособность эндпоинта - просто нужно взять пример запроса для этого эндпоинта и отослать его на сервер. Если у эндпоинта есть параметры, то их значения нужно также взять из примеров.  Пришедший ответ сравнить с примером ответа или проваледировать его json-схему на основании документации. Для того, чтобы примеры ответов соответствовали ответам сервера, сервер должен работать с правильными данными в БД. Таким образом, при наличии БД с тестовыми данными и документации api сервиса с описанием ответов и примерами запросов, мы можем обеспечить простое тестирование работоспособности нашего сервиса. О нашей документации, системе тестирования и БД сейчас я упомянул для полноты картины и мы о них обязательно поговорим в другой раз. Сегодня же речь пойдет о том, как на основании такой документации генерировать как можно больше полезного серверного кода.

Наш сервер написан на play 2.5 и предоставляет rest api своим клиентам. Формат обмена данными - json. Стандартное описание api в play framework происходит в файле `conf/route`. Синтаксис этого описания прост и ограничивается описанием имен эндпоинтов и их параметров, а также привязкой эндпоинтов к методам контроллера в файле routs. Нашей целью будет замена стандартного синтаксиса на описание в формате RAML. Для этого нам нужно:

1. разобраться как в play устроена маршрутизация и как обрабатываются route файлы
2. заменить стандартный механизм маршрутизации на наш механизм, использующий raml
3. посмотреть на результат и сделать выводы :)

Итак, давайте по порядку.

##Роутинг в play framework

Play framework рассчитан на использование с двумя языками - scala так и java. Поэтому для описания маршрутов авторы фреймворка не стали использовать dsl на базе какого-то конкретного языка, а написали свой язык и компилятор к нему. Далее я буду говорить про scala, но все сказанное справедливо и для java. 
Play собирается [sbt](http://www.scala-sbt.org/). Во время сборки проекта route файлы компилируются в файлы на scala или java и результат компиляции используется далее при сборке. За обработку route файла отвечает sbt-плагин `com.typesafe.play.sbt-plugin`. Давайте посмотрим как он работает. Но для начала пару слов об sbt.

Основным понятием sbt является ключ. Ключи бывают двух типов `TaskKey` и `SettingsKey`. Первый тип используется для хранения функций. Каждое обращение к этому ключу приводит к вызову этой функции. Второй тип ключа хранит константу и вычисляется один раз. `compile` - это `TaskKey`, в процессе выполнения он вызывает другой `TaskKey`, `sourceGenerators`, для кодогенерации и создания исходных файлов. Собственно sbt-plugin добавляет функцию обработки route файла к `sourceGenerators`.

Обычно на основе route создается два основных артефакта - файл `target/scala-2.11/routes/main/router/Routes.scala` и `target/scala-2.11/routes/main/controllers/ReverseRoutes.scala`. Класс `Routes` используется для маршрутизации входящих запросов. `ReverseRoutes` используется для вызова эндпоинтов из кода контроллеров и view по имени эндпоинта. Давайте проиллюстрируем вышесказанное примером.

conf/routes:

	GET     /test/:strParam   @controllers.HomeController.index(strParam)

Тут мы объявляем параметризованный эндпоинт и мапим его на метод `HomeController.index`. В результате компиляции этого файла получается следующий код на scala.

target/scala-2.11/routes/main/router/Routes.scala:

{% highlight scala %}

class Routes(
	override val errorHandler: play.api.http.HttpErrorHandler, 
	HomeController_0: javax.inject.Provider[controllers.HomeController],
	val prefix: String
) extends GeneratedRouter {
	
	...

	private[this] lazy val controllers_HomeController_index0_route = Route("GET", 
		PathPattern(List(
			StaticPart(this.prefix), 
			StaticPart(this.defaultPrefix), 
			StaticPart("test/"), 
			DynamicPart("strParam", """[^/]+""",true)
		))
		)

	private[this] lazy val controllers_HomeController_index0_invoker = createInvoker(
		HomeController_0.get.index(fakeValue[String]),HandlerDef(
			this.getClass.getClassLoader,
			"router","controllers.HomeController","index",
			Seq(classOf[String]),"GET","""""",this.prefix + """test/""" + "$" + """strParam<[^/]+>""")
		)

	def routes: PartialFunction[RequestHeader, Handler] = {
		case controllers_HomeController_index0_route(params) =>
			call(params.fromPath[String]("strParam", None)) { (strParam) =>
				controllers_HomeController_index0_invoker.call(HomeController_0.get.index(strParam))
			}
		}
}

{% endhighlight %}

Этот класс занимается маршрутизацией входящих запросов. В составе аргументов ему передаются ссылка на контроллер (точнее инжектор, но это не существенно) и префикс url пути, который настраивается в конфигурационном файле. Далее в классе объявлена "маска" маршрутизации `controllers_HomeController_index0_route`. Маска состоит из HTTP глагола и паттерна маршрута. Паттерн маршрута состоит из частей, каждая соответствует элементу url пути. `StaticPart` определяет маску для неизменной части пути, `DynamicPart` - задает шаблон для url параметра. Каждый входящий запрос попадает в функцию `routes`, где сопоставляется с доступными масками (в нашем случае она одна). Если совпадений не найдено - клиент получит 404 ошибку, в противном случае будет вызван соответствующий обработчик. В нашем примере обработчик один - это `controllers_HomeController_index0_invoker`. В обязанности обработчика входит вызов метода контроллера с нужным набором параметров и трансформация результатов этого вызова. 

target/scala-2.11/routes/main/controllers/ReverseRoutes.scala:

{% highlight scala %}

package controllers {
	class ReverseHomeController(_prefix: => String) {
		...

		def index(strParam:String): Call = {
			import ReverseRouteContext.empty
			Call("GET", _prefix + { _defaultPrefix } + 
				"test/" + 
				implicitly[PathBindable[String]].unbind("strParam", dynamicString(strParam)))
		}
	}
}

{% endhighlight %}

Данный код позволяет нам обращаться к эндпоинту через соответствующую функцию, полезно, например во view.

Итак, чтобы сменить формат описания маршрутов нам достаточно написать свой генератор файла `Routes`. `ReverseRoutes` нам не нужен, так как наш сервис отдает json и view у него нет. Чтобы наш генератор сработал мы должны включить его. Можно копировать исходники генератора в каждый проект, где он нужен, далее подключать его в `build.sbt`. Но более правильно будет оформить генератор в виде плагина к sbt.

##Плагин SBT

О плагинах sbt исчерпывающе [написано](http://www.scala-sbt.org/0.13/docs/Using-Plugins.html) в документации. Тут я упомяну об основных, на мой взгляд, моментах. Плагин - это набор дополнительной функциональности для sbt. Обычно плагины добавляют в проект новые ключи и расширяют существующие. Нам, например, нужно будет расширить ключ `sourceGenerators`. Одни плагины могут зависеть от других, например мы могли бы использовать в качестве основы плагин `com.typesafe.play.sbt-plugin` и изменить в нем только то, что нам нужно. Другими словами наш плагин зависит от `com.typesafe.play.sbt-plugin`. Чтобы sbt автоматически подключал все зависимости для нашего плагина, наш плагин должен быть [AutoPlugin'ом](http://www.scala-sbt.org/0.13/docs/Using-Plugins.html#Enabling+and+disabling+auto+plugins). Ну и последнее. Из-за вопросов совместимости плагины пишутся на scala 2.10.

Итак, нам нужно генерировать `Routes.scala` на основе файла RAML. Пусть этот файл называется `conf/api.raml`. Для того, чтобы документацию в RAML формате можно было использовать для маршрутизации, необходимо каким-то способом указать в нем для каждого эндпоинта метод контроллера, который необходимо вызвать при приходе запроса. RAML 0.8, который мы будем использовать, не имеет средств для указания такой информации, поэтому придется делать грязный хак (RAML 1.0 решает эту проблему с помощью аннотаций, но на момент написания статьи эта версия стандарта еще сыра). Добавим информацию о вызываемом методе контроллера в первую строку discription для каждого эндпоинта. Наш пример из прошлого раздела в RAML формате будет выглядеть так:

{% highlight yaml %}

/test/{strParam}:
	uriParameters:
		strParam:
		description: simple parameter
		type: string
		required: true
		example: "some value"
	get:
		description: |
			@controllers.HomeController.index(strParam)
		responses:
			200:
				body:
					application/json:
						schema: !include ./schemas/statements/operations.json
						example: !include ./examples/statements/operations.json

{% endhighlight %}

На деталях парсинга RAML останавливаться не буду, скажу лишь что можно использовать [парсер от raml.org](https://github.com/raml-org/raml-java-parser). В результате парсинга мы получаем список правил - по одному на каждый эндпоинт. Правило задается следующим классом:

{% highlight scala %}

	case class Rule(verb: HttpVerb, path: PathPattern, call: HandlerCall, 
		comments: List[Comment] = List())

{% endhighlight %}

Названия и типы полей говорят сами за себя. Теперь для каждого правила мы можем в файле `Routes.scala` создать свою маску, обработчик и элемент `case` в функции route. Для решения этой задачи мы могли бы просто руками генерировать строку с кодом `Routes.scala` на основе списка правил или применить макросы. Но лучше выбрать промежуточный вариант, который предпочли и разработчики play - использовать шаблонизатор. play использует шаблонизатор [twirl](https://github.com/playframework/twirl) и мы тоже его используем. Вот шаблон из нашего плагина, генерирующий функцию `route`:

{% highlight scala %}

def routes: PartialFunction[RequestHeader, Handler] = @ob
	@if(rules.isEmpty) {
	Map.empty
	} else {@for((dep, index) <- rules.zipWithIndex){@dep.rule match {
	case route @ Rule(_, _, _, _) => {
		case @(routeIdentifier(route, index))(params) =>
			call@(routeBinding(route)) @ob @tupleNames(route)
				@paramsChecker(route) @(invokerIdentifier(route, index))
					.call(@injectedControllerMethodCall(route, dep.ident, x => safeKeyword(x.name)))@cb
		}
	}}}@cb

{% endhighlight %}

Выглядет несколько запутанно, но если присмотреться, то все становится ясно. Выражения, начинающиеся с `@` - это директивы и переменные шаблонизатора. Так переменные `@ob` и `@cb` будут раскрыты в `{` и `}` соответственно. А, например, `@(routeIdentifier(route, index))` развернется по следующему правилу:

{% highlight scala %}

def routeIdentifier(route: Rule, index: Int): String = route.call.packageName.replace(".", "_") + 
	"_" + route.call.controller.replace(".", "_") + 
	"_" + route.call.method + index + "_route"

{% endhighlight %}

Теперь ясно как написать код, создающий `Routes.scala` на основе RAML и понятно как подключить его к сборке. [Исходники](https://github.com/bavadim/raml2play) готового плагина лежат на гитхабе.

##Планы на будущее

Плагин позволил нам использовать документацию в качестве исходного кода для сервера. Но кодогенерация не использует всей доступной информации из RAML файла. А именно мы никак не используем информацию о типе запроса и ответа. В play парсинг запроса и генерация ответа происходит в методах контроллера, но мы хотим генерировать этот код автоматически. Кроме того у нас в планах использовать версию RAML 1.0. 

На сегодня все, спасибо за внимание!
