---
layout: post
title: "Про radio buttons и ModelForm в Django"
date: 2011-04-14 11:31
comments: true
categories: 
---
Допустим есть задача в форме давать возможность человеку указать свой пол. Конкретно для пола нам хватит поля django.forms.BooleanField или django.forms.NullBooleanField, если указывать пол не обязательно. Но поскольку данный пример взят в образовательных целях, а в реальной жизнии скорее всего будет больше, чем 2 варианта выбора, будем использовать django.forms.IntegerField.

Например у нас есть такая модель

{% codeblock lang:python %}
GENDER_MALE = 0
GENDER_FEMALE = 1
CHOICES_GENDER = ((GENDER_MALE, _("Male")), 
                  (GENDER_FEMALE, _("Female")))

class Men:
    gender = forms.IntegerField(_("Gender"),
                                choices=CHOICES_GENDER)
{% endcodeblock %}

И такая форма

{% codeblock lang:python %}
class MenForm(forms.ModelForm):
    class Meta:
        model = Men
        widgets = {'gender': forms.RadioSelect()}
{% endcodeblock %}

Если мы попробуем вывести в шаблоне эту форму, то обнаружим не два radio-button'а, а три. Первым будет пустой выбор. Давайте попробуем разобраться почему.
<!--more-->

При создании полей для формы, вызывается метод *django.forms.models.fields_for_model*. В этот метод передаются параметры, которые вы могли бы описать в классе *Meta* класса *ModelForm*: *model, fields, exclude, widgets, formfield_callback*.

В случае, если formfield_callback не Null, будет вызван этот метод для получения инстанса django.forms.IntegerField. В общем же случае, для каждого поля нашей модели вызывается метод *formfield*. Т.е. для получения объекта класса *django.forms.IntegerField* мы вызовем *model.fields["gender"].formfield()*, который вернет там IntegerField с нужным нам виджетом (RadioSelect).

Посмотрим что происходит в методе formfield и где берется дополнительный элемент в списке choices. В этом методе есть такие строки

{% codeblock lang:python %}
include_blank = self.blank or not (self.has_default() or 'initial' in kwargs)
defaults['choices'] = self.get_choices(include_blank=include_blank)
{% endcodeblock %}

Как понятно из названия переменной, если include_blank==True, то в choices добавляется "путое значение". Попробуем разобраться для чего это сделано внимательно вчитавшись в первую строчку. Итак,

- если поле может быть пустым (self.blank==True), то добавить пустой поле
- если нет значения по умолчанию у этого поля модели, либо если при создании формы не было передано initial значение для этого поля, то также добавляется пустое поле

С причиной мы разобрались. Подумаем теперь как добиться того, что нам нужно.

### Вариант номер один

Наиболее простой вариант &mdash; это добавить значение по умолчанию в модель.

{% codeblock lang:python %}
GENDER_MALE = 0
GENDER_FEMALE = 1
CHOICES_GENDER = ((GENDER_MALE, _("Male")), 
                  (GENDER_FEMALE, _("Female")))

class Men:
    gender = forms.IntegerField(_("Gender"),
                                choices=CHOICES_GENDER,
                                default=GENDER_MALE)
{% endcodeblock %}

Недостаток данного метода, что у пользователя всегда будет выбрано одно из значений, и мы не заставим его внимательно прочитать что от него хотят и в большинстве случаев будем получать значение по умолчанию.

### Метод номер два

Можно переопределить значение поля gender, которое создается метаклассом ModelForm. В этом случае forms.py у вас будет выглядеть примерно так.

{% codeblock lang:python %}
class MenForm(forms.ModelForm):
    gender = forms.IntegerField(widget=forms.RadioSelect(choices=CHOICES_GENDER))

    class Meta:
        model = Men
{% endcodeblock %}

Явных недостатков не вижу, кроме как необходимо отдельно определять label для поля и др. значения, которые уже не возьмутся автоматически с определения модели.

### Метод номер три

Ну и на закуску самый "хакерский" метод. Как вы помните, функция fields_for_model в качетсве параметра принимает formfield_callback &mdash; это метод, который будет вызываться для создания всех полей формы. Вызываться этот метод будет вместо formfield. Не буду дого рассказывать откуда что берется и как происходит внутри, сразу приведу пример. Подробности можно прочитать в файле django/forms/models.py

{% codeblock lang:python %}
def formfield_cb(field, **kwargs):
    if field.name == 'gender':
        kwargs["choices"] = CHOICES_GENDER
    return field.formfield(**kwargs)

class MenForm(forms.ModelForm):

    formfield_callback = formfield_cb

    class Meta:
        model = Men
        widgets = {'gender': forms.RadioSelect(choices=CHOICES_GENDER)}
{% endcodeblock %}

Ну и традиционно про недостатки. Уж очень как-то много букв нужно писать чтоб все заработало. Ну и строчку *formfield_callback = formfield_cb* нужно включать во всех наследников базовой формы. Если мы унаследуем форму и забудем добавить эту строчку, то новая форма снова будет иметь пустое значение в radio-select виджете.
