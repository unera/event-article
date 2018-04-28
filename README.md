# Именованные события: программируем GUI

В настоящее время большинство сайтов представляют собой по сути обычную программу, которая обрабатывает те или иные действия пользователей. Для того чтобы упрощать жизнь программистам реализованы десятки фреймворков помогающих решать те или иные задачи.

Если рассматривать программирование GUI или UI вообще, то в обобщенном случае UI представляет собой множество **слабосвязанных** задач в одном пакете. 

Например раздел "погода" на главной страничке поисковика - является просто индикатором. Выбрав тему - мы можем увидеть дождик или солнышко на фоне. Нужно ли устанавливать взаимосвязь между разделом "погода" и фоновой картинкой? С точки зрения минимизации компьютерных расходов - безусловно: хорошо повторно использовать данные полученные однажды. Однако с точки зрения разработки программирование слабосвязанных вещей может тянуть за собой настолько большие трудозатраты, что иногда проще отказаться от связанности и два раза запросить одни и те же данные.

О программировании слабосвязанных вещей попробуем поговорить в данной статье.

</cut>

Сразу обращаю внимание, что данная статья ставит целью **описать подход к решению** задачи, поэтому некоторая часть кода приведённого в статье может выглядеть "велосипедно", либо Вы можете указать ссылку на более профессиональную реализацию.



Итак, поскольку мы с самого начала статьи сформулировали термин "слабосвязанный", то логично будет поисследовать уровень связи в разных подходах.


Отвлечемся от программирования.

Вам необходимо уведомить Вашего знакомого о том какая погода стоит нынче. У Вас есть множество путей. Выберем основные и расположим их в порядке ослабления взаимосвязи, Вы можете:

1. Позвонить знакомому и сообщить ему нужную информацию.
2. Отправить ему сообщение электронной почтой.
3. Написать о погоде в соцсети. Кому надо - прочитает :)

Если вернуться к программированию, то перечисленное будет представлять собой:

1. Обычный ООП/функциональный стиль: вызов метода.
2. Отправка сообщения от объекта к объекту.
3. Отправка широковещательного сообщения.


Данная статья целиком посвящена третьему варианту.


Если мы обратимся из мира веб-программирования в мир обычного GUI, например - операционные системы, то увидим что там сталкиваются с тем же набором проблем.

В каждой операционной системе существует решение, позволяющее связывать между собой слабосвязанные вещи. В Linux этим решением является dBus.
Как это работает? Когда пользователь закрывает крышку ноутбука или когда сетевой интерфейс производит подключение к внешнему миру, то в шину dBus отправляется сообщение.

Теперь коду, который работает с крышкой ноутбука не нужно иметь связь со всеми связанными компонентами (которые могут присутствовать, могут отсутствовать), а достаточно сообщать в системную шину данные о себе. Множество устройств наполняют системную шину сообщениями:

- (драйвер крышки) если кому-то интересно, то крышка ноутбука открыта!
- (сетевая карта) возможно это неинтересно никому, но соединение с внешним миром установлено!
- (плеер) я здесь показываю фильм пользователю!
- (датчик температуры) у меня 69 градусов!
- (драйвер USB) у меня тут флешку воткнули!
- (плеер) я все еще показываю фильм пользователю!

Несколько утрировано и несколько оторвано от реальности, но наглядно :)

Теперь если мы хотим понимания что происходит в системе, мы просто слушаем эту шину. А если хотим интегрироваться с системой, то начинаем и отправлять данные в эту шину.


Вернемся к вебпрограммированию и попробуем реализовать аналогичную шину.
Прежде всего определимся с сообщениями. Сообщение имеет тип (идентификатор) и произвольные данные.

Давайте выберем простой API. Отправлять сообщения будем например так:

```js

EV('погода', { t: 10 });

```

Теперь в коде, который принимает погоду по AJAX просто впишем после успешного ее приема:

```js

$.ajax({
   url: 'http://погода-по-ajax.bla',
   success: function(data) {
	// прочий код
	EV('погода', data);
   }
});
```

Все, на этом интеграция источника данных с внешним миром завершена.

Заинтересованный в погоде код может выглядеть следующим образом:

```js

EV.on('погода', function(data) {
	if ('t' in data) {
		EV('погода-температура', data.t);
	}
	if ('wind' in data) {
		EV('погода-ветер', data.wind);
	}
});

```

Как видим принявший данные код может тоже отправлять события.

Ну и далее, если у нас например есть уголок где надо показать температуру, то мы вешаем обработчик:

```js

EV.on('погода-температура', function(t) {
	$('#my-temp').text(t);
	$('#my-temp').removeClass('hidden');
});

```


Библиотека EV может выглядеть примерно так:

```js

window.EV = function(name) {
	var s = window.EV._list;
	if (name in s) {
		for (var i in s[name]) {
			s[name][i].apply(null, arguments);
		}
	}
};

window.EV._list = {};		// подписчики

window.EV.on = function(name, cb) {
	if (!(name in window.EV._list))
		window.EV._list[name] = [];
	window.EV._list[name].push(cb)

}

```

Обработку исключений в обработчиках можно добавить по вкусу.

Взяв на борт сайта подобную библиотеку обнаруживаем что помимо связности, некоторые сложные ранее проблемы становятся более простыми.

Например реакция на клики по элементам пришедшим по AJAX:

```js

$('body').on('click', '[data-event]', function() {
	EV($(this).attr('data-event'), $(this).attr('data-event-arg'), $(this));
	return false;
});

```


