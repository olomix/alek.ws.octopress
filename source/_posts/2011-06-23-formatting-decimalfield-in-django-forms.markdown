---
layout: post
title: "Форматируем DecimalField в Django Forms"
date: 2011-06-23 14:41
comments: true
categories: 
---
В статье рассмотрены две задачи

- Как выводить forms.DecimalField в нужном вам формате (например, обрезать незначащие нули)
- Как переопределять поля по умолчанию для ModelForm не переписывая каждое поле в отдельности

Рассмотрим задачу создания и редактирования модели, служащей для снятия антропометрии человеческого тела. Эта модель была использована при создании веб-приложения для спортивного клуба, где клиенты могут снимать с себя замеры и наблюдать за динамикой изменения своего тела. Возьмем упрощенный вариант, где нужно замерять объем груди, талии и вес. В реальности параметров гораздо больше, но для примера хватит.
<!--more-->

{% codeblock lang:python %}
msrmnt_attrs = {'max_digits': 5, 
                'decimal_places': 1,
                'blank': True,
                'null': True}

class Measurement(models.Model):
    user = models.ForeignKey('auth.User')
    date = models.DateField(_('Date'))
    weight = models.DecimalField(_('Weight'), **msrmnt_attrs)
    chest = models.DecimalField(_('Chest'), **msrmnt_attrs)
    waist = models.DecimalField(_('Waist'), **msrmnt_attrs)
{% endcodeblock %}

И, соответственно, форма

{% codeblock lang:python %}
class MeasurementForm(forms.ModelForm):
    class Meta:
        model = Measurement
{% endcodeblock %}

Замеры должны сниматься в сантиметрах, но некоторые особо ответственные личности желают записывать еще и десятые доли сантиметра. Например, объем груди может составлять 105,5 см., а не 105 или 106. Нам нужно сделать хорошо всем — и тем кто умеет аппроксимировать до целых чисел, и людям (назовем их культурно), любящих точность.

Можно было использовать PositiveIntegerField и сохранять параметры в миллиметрах. Или FloatField и как-то обходить машинную погрешность когда 0.1 + 0.1 + 0.1 - 0.3 = 5.5511151231257827e-017. Но умные люди придумали для нас модуль [decimal](http://docs.python.org/library/decimal.html) в Python, а другие люди сделали поле [DecimalField](https://docs.djangoproject.com/en/dev/ref/models/fields/#decimalfield) в Django специально для этих целей.

Использовав DecimalField мы сразу натыкаемся на проблему при попытке ввести запятую в качестве разделителя дробной части. По умолчанию, формы в Django не слушаются текущей локали и хотят получать числа с точкой, в качестве разделителя. Это решается добавлением localize=True в качестве параметра конструктору forms.DecimalField.

Следующее, мы хотим получать от пользователя только положительные числа. В этом случае нам поможет параметр min_value=0. И еще мы хотим принимать после запятой только один знак. Снимать замеры в долях миллиметров это уже слишком. Для этого служат параметры max_digits=5 и decimal_places=1. Поскольку каждый замер является не обязательным, то добавим к имеющимся у нас параметрам blank=True и null=True.

И последняя незадача. Если мы введем число 105 и сохраним его. То при попытке отредактировать, получим уже 105,0 а не 105. Это специфика работы DecimalField. Вроди мелочь, а неприятно.

Чтоб победить последнюю проблему, нам нужно понять как работают Form и ModelForm. Класс Form берет свои данные, которые он позволяет нам редактировать, из двух источников. Первое, мы можем их задать в параметре initial при создании формы

{% codeblock lang:python %}
form = MeasurementForm(initial={"field1": 1, "field2": 2})
{% endcodeblock %}

И второе, мы передаем данные в параметре data при сохранении (ну и files для соответствующих полей)

{% codeblock lang:python %}
form = MeasurementForm(initial={"field1": 1, "field2": 2}, 
                       data=request.POST)
{% endcodeblock %}

Класс ModelForm пользуется именно параметром initial. ModelForm является наследником Form. Когда мы пытаемся отредактировать некую модель, то создаем ее форму следующим образом

{% codeblock lang:python %}
form = MeasurementForm(instance=measurement)
{% endcodeblock %}

Внутри это все выглядит как получение текущих значений полей модели measurement и передача их в качестве параметра initial конструктору суперкласа.

Механизм работы ModelForm был рассмотрен для понимания где форма хранит свои данные, которые мы потом видим на странице. Либо в data — это данные, которые мы только что ввели, они не прошли валидацию и отображаются для нас в том виде, в котором мы их задали. Либо в initial — этот параметр заполняется конструктором ModelForm. Именно сюда и попадает наше число 105,0, которое нам нужно заменить на 105.

Когда мы пытаемся получить доступ к данным нашей формы, для каждого значения вызывается метод Field.prepare_value. Т.е., если initial = {"weight": 105.0}, то при выводе данной формы в шаблоне на определенном этапе будет вызван метод DecimalField.prepare_value(105.0). По умолчанию метод возвращает полученный параметр, ничего с ним не делая. В этом месте мы и можем исправить ситуацию и подменить число на 105, вместо 105,0. Создадим новый класс, унаследовав его от DecimalField и переписав prepare_value.

{% codeblock lang:python %}
class RoundedDecimalField(forms.DecimalField):
    def prepare_value(self, value):
        if isinstance(value, decimal.Decimal):
            return value.normalize()
        return super(RoundedDecimalField, self).prepare_value(value)
{% endcodeblock %}

Зачем здесь сделана проверка является ли value инстансом класса decimal.Decimal? Для случая когда мы сохраняем форму и значение берется из data (request.POST), а не из initial. А в request.POST хранятся unicode string, а не decimal.Decimal. И у этого класса нет метода normalize. Тогда мы просто передаем значение методу суперкласса, которые вернет его не меняя.

Итак мы подошли к финалу и имеем следующее. Нам нужна форма, построенная из модели Measurement и имеющая в качестве поля для models.DecimalField класс RoundedDecimalField, а не forms.DecimalField. Причем в качестве дополнительных параметров для этого поля должны передаваться localize=True и min_value=0. Если следовать официальной документации, то мы должны переопределить все такие поля в форме нужными нам классами. Т.е. сделать примерно следующее.

{% codeblock lang:python %}
msrmnt_attrs = {'max_digits': 5, 
                'decimal_places': 1,
                'blank': True,
                'null': True,
                'min_value': 0,
                'localize': True,}

class MeasurementForm(forms.ModelForm):
    weight = RoundedDecimalField(label=_('Weight'), **msrmnt_attrs)
    chest = RoundedDecimalField(label=_('Chest'), **msrmnt_attrs)
    waist = RoundedDecimalField(label=_('Waist'), **msrmnt_attrs)

    class Meta:
        model = Measurement
{% endcodeblock %}

Если у нас несколько десятков полей, то вообще стает грустно переписывать по второму разу label и создавать новое поле для каждого элемента модели Measurement, когда это может делать за нас ModelForm.

На самом деле, мы можем избежать всего этого. Внутри MedelForm вызывает определенный метод, который создает forms.Field для каждой models.Field. По умолчанию, каждый наследник класса models.Field имеет метод formfield, который создает для себя поле формы. Т.е. мы можем написать метод, который для полей DecimalField будет создавать нужные нам поля формы класса RoundedDecimalField, а для всех остальных просто вызывать метод formfield для создания полей по умолчанию. Этот метод называется formfield_callback и активно используется в приложении django.contrib.admin, где для многих полей моделей это приложение переопределяет виджеты по умолчанию на свои.

Проще всего это будет видно на примере. Вот так будет выглядеть файл forms.py не зависимо от того, сколько полей DecimalField включает наша модель

{% codeblock lang:python %}
class RoundedDecimalField(forms.DecimalField):
    def prepare_value(self, value):
        if isinstance(value, decimal.Decimal):
            return value.normalize()
        return super(RoundedDecimalField, self).prepare_value(value)

msrmnt_attrs = {'min_value': 0,
                'localize': True,
                'form_class': RoundedDecimalField}

def msrmnt_formfield_callback(f, **kwargs):
    if isinstance(f, models.DecimalField):
        kwargs.update(msrmnt_attrs)
    return f.formfield(**kwargs)

class MeasurementForm(forms.ModelForm):
    formfield_callback = msrmnt_formfield_callback

    class Meta:
        model = Measurement
{% endcodeblock %}

Как видно, мы не дублировали ни одного параметра, которые уже заданы в моделе (max_digits, required и т.д.). И нам не нужно переопределять каждое поле модели чтоб получить заветный результат.

Вывод. Django имеет прекрасную документацию, но если следовать только ей, и не изучать исходные тексты, то в некоторых случаях может быть реально грустно.
