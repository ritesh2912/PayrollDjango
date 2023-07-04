# Track Time Hours and Salary with Django
Hello, world. In this article, we will create a Django app, which will handle employers and we will track their hours of work and payments. 
We will use the next frameworks and libraries to help us create that. 
( I assume you have already installed python and you know the basics steps)

Django Framework ==> https://www.djangoproject.com/
Bootstrap 4 ==> https://getbootstrap.com/. You get here the starter template, which contains jquery and the Bootstrap theme, CSS etc
And xdsoft datetime picker ==> https://xdsoft.net/jqplugins/datetimepicker/
Part 1. Create the App

After installing Django or I assume you have it, let’s create our app.

$ django-admin start project payroll_warehouse
The next step create our apps.

$ cd payroll_warehouse
$ python manage.py startapp payroll
$ python manage.py startapp frontend
The front-end app will place our views, templates, and static files and the payroll app will add our models, forms etc..

Inside the frontend folder let’s create two more folders with the names templates and static, so the directory will be like this.


Inside the static folder add these files ==> https://github.com/Zefarak/payroll_blog_app/tree/master/frontend/static/datetimepicker

Inside the templates folder add these files ==> https://github.com/Zefarak/payroll_blog_app/tree/master/frontend/templates

Will explain om last part of the frontend.

Now it’s time to edit setting.py files inside the payroll_warehouse folder. I will add only the changes, you can always grab the whole file from github.

INSTALLED_APPS = [
    'django.contrib.admin',
    'django.contrib.auth',
    'django.contrib.contenttypes',
    'django.contrib.sessions',
    'django.contrib.messages',
    'django.contrib.staticfiles',

    'payroll',
    'frontend',
]
First we register our apps.

STATIC_URL = '/static/'
STATIC_ROOT = 'static'

CURRENCY = '€'
And then we inform the django app where the static folder is and adding the currency variable.

The second file need edit is the urls.py. Delete anything from this file and add this.

from django.contrib import admin
from django.urls import path

from frontend.views import (HomepageView, create_occupation_view, create_person_view, delete_person_view,
                            occupation_delete_view, PersonCardView, OccupationUpdateView,
                            handle_payroll_view, handle_schedule_view
                            )

urlpatterns = [
    path('admin/', admin.site.urls),
    path('', HomepageView.as_view(), name='homepage'),
    path('create-peron/', create_person_view, name='create_person'),
    path('create-occupation/', create_occupation_view, name='create_occupation'),
    path('update-occupation/<int:pk>/', OccupationUpdateView.as_view(), name='update_occup'),
    path('delete-occupation/<int:pk>/', occupation_delete_view, name='delete_occup'),
    path('person-card/<int:pk>/', PersonCardView.as_view(), name='person_card'),
    path('person/create-payroll/<int:pk>/<slug:type_>/', handle_payroll_view, name='handle_payroll'),
    path('person/create-schedule/<int:pk>/<slug:type_>/', handle_schedule_view, name='handle_schedule'),
    path('person/delete/<int:pk>/', delete_person_view, name='person_delete')
]
Here just copy the code, this code include the views we will write and with the help of path we match them with the url. Further explain later.

Done let’s move on.

Part 2. Payroll Folder, Create The models.

First, we will create some abstract models. So inside the payroll folder we create a new file called abstract_models.py.

from django.db import models
from django.contrib.auth.models import User
from django.utils import timezone
from django.conf import settings
import uuid
CURRENCY = settings.CURRENCY

import datetime
from dateutil.relativedelta import relativedelta
What does abstract models? We use them as a base model to create different models with same fields and little changes if needed so we don’t repeat ourselves.

Let’s add here a function too.

def initial_date(request, months=12):
    #  gets the initial last three months or the session date
    date_now = datetime.datetime.today()
    current_year = f'01/01/{datetime.date.today().year} - 12/31/{datetime.date.today().year}'
    date_range = request.GET.get('date_range', current_year)
    date_start, date_end = None, None

    if date_range:
        try:
            date_range = date_range.split('-')
            date_range[0] = date_range[0].replace(' ','')
            date_range[1] = date_range[1].replace(' ','')
            date_start = datetime.datetime.strptime(date_range[0], '%m/%d/%Y')
            date_end = datetime.datetime.strptime(date_range[1],'%m/%d/%Y')
        except:
            print('except hitted')
            date_three_months_ago = date_now - relativedelta(months=months)
            date_start = date_three_months_ago
            date_end = date_now
            date_range = '%s - %s' % (str(date_three_months_ago).split(' ')[0].replace('-','/'),str(date_now).split(' ')[0].replace('-','/'))
            request.session['date_range'] = '%s - %s'%(str(date_three_months_ago).split(' ')[0].replace('-','/'),str(date_now).split(' ')[0].replace('-','/'))
    return [date_start, date_end, date_range]
What does? Get’s the date from frontend format and transform to a format that django understand so he can use it.

class DefaultOrderModel(models.Model):
    uid = models.UUIDField(default=uuid.uuid4, editable=False)
    title = models.CharField(max_length=150)
    timestamp = models.DateTimeField(auto_now_add=True)
    edited = models.DateTimeField(auto_now=True)
    notes = models.TextField(blank=True, null=True)
    date_expired = models.DateField(default=timezone.now)
    value = models.DecimalField(decimal_places=2, max_digits=20, default=0)
    taxes = models.DecimalField(decimal_places=2, max_digits=20, default=0)
    paid_value = models.DecimalField(decimal_places=2, max_digits=20, default=0)
    final_value = models.DecimalField(decimal_places=2, max_digits=20, default=0)
    discount = models.DecimalField(decimal_places=2, max_digits=20, default=0)
    is_paid = models.BooleanField(default=True)
    printed = models.BooleanField(default=False)
    objects = models.Manager()

    class Meta:
        abstract = True

    def __str__(self):
        return self.uid

    def tag_is_paid(self):
        return 'Is Paid' if self.is_paid else 'Not Paid'

    def tag_value(self):
        return f'{self.value} {CURRENCY}'
    tag_value.short_description = 'Αρχική Αξία'

    def tag_final_value(self):
        return f'{self.final_value} {CURRENCY}'
    tag_final_value.short_description = 'Αξία'

    def tag_paid_value(self):
        return f'{self.paid_value} {CURRENCY}'

    def get_remaining_value(self):
        return self.final_value - self.paid_value

    def tag_payment_method(self):
        return f'{self.payment_method} {CURRENCY}'
This is a default order model. Many features will be not used, just added as a example, like the paid function or printed. So if we had a bigger app we could this model as a Sale Model, Warehouse Invoice, in fact anything we want, and make our life easier.

Now let’s go to models.py .

from .abstract_models import *

from django.shortcuts import reverse
from django.db import models
from django.db.models import Sum
from django.contrib.auth import get_user_model
from django.contrib.contenttypes.models import ContentType
from django.conf import settings

User = get_user_model()

CURRENCY = settings.CURRENCY
PAYROLL_CHOICES = (
    ('1', 'Salary'),
    ('2', 'Extra'),
    )
First the imports, nothing fancy here.

class Occupation(models.Model):
    active = models.BooleanField(default=True)
    title = models.CharField(max_length=64)
    notes = models.TextField(blank=True, null=True)
    balance = models.DecimalField(max_digits=50, decimal_places=2, default=0)

    objects = models.Manager()

    def __str__(self):
        return self.title

    def save(self, *args, **kwargs):
        self.balance = self.person_set.all().aggregate(Sum('balance'))['balance__sum'] \
            if self.person_set.all().exists() else 0
        super().save(*args, *kwargs)

    def tag_balance(self):
        return '%s %s' % (self.balance, CURRENCY)

    tag_balance.short_description = 'Remaining'
The Occupation model will be used for filtering the Employers. So the fields we need is, check if it’s active, the title , some notes if needed and the balance is a meter that gets all the remaining money you need to pay per occupation. To achieve that we need to overwrite the save method of the model. Let’s see it. (again maybe i will not use anything)

def __str__(self):
    return self.title

def save(self, *args, **kwargs):
    self.balance = self.person_set.all().aggregate(Sum('balance'))['balance__sum'] \
        if self.person_set.all().exists() else 0
    super().save(*args, *kwargs)
No need to explain the __str__method, let’s see the save method now. When a save trigger on a occupation object, this calculation always run to refresh the balance. Step by step, we get all the employers(persons), related in this object and with the help of aggregate and Sum we get the desire result. We always need to check if there is a queryset because if there no objects, the program will crash.

Now is time to Create the Person model.

class Person(models.Model):
    active = models.BooleanField(default=True)
    timestamp = models.DateTimeField(auto_now_add=True)
    edited = models.DateTimeField(auto_now=True)
    title = models.CharField(max_length=64, unique=True)
    phone = models.CharField(max_length=10, blank=True)
    phone1 = models.CharField(max_length=10, blank=True)
    occupation = models.ForeignKey(Occupation, null=True, blank=True, verbose_name='Απασχόληση', on_delete=models.PROTECT)
    balance = models.DecimalField(max_digits=50, decimal_places=2, default=0)
    value_per_hour = models.DecimalField(max_digits=20, decimal_places=2, default=0)
    extra_per_hour = models.DecimalField(max_digits=20, decimal_places=2, default=0)
    monthly_salary = models.DecimalField(max_digits=20, decimal_places=2, default=0)

    objects = models.Manager()
In this model will add the employers. What info we need? We have to check if the person is active, when added or edited, his name, phone, which occupation belongs for easy filtering and his balance and some salary.W will have 2 different salary options, one for extra time and one for normal

After that lets add this methods.

def save(self, *args, **kwargs):
    self.balance = self.update_balance()
    super().save(*args, **kwargs)
    self.occupation.save() if self.occupation else ''

def update_balance(self):
    queryset = self.person_invoices.all()
    value = queryset.aggregate(Sum('final_value'))['final_value__sum'] if queryset else 0
    paid_value = queryset.aggregate(Sum('paid_value'))['paid_value__sum'] if queryset else 0
    diff = value - paid_value
    return diff
We override the save method again, this time to update the balance we call the update_balance function, which works like before with occupation, we save the object, and after that we save the occupation if exists so we trigger the save method we explain earlier. Now update_balance method. First step we get all the invoices for this person and calculate the total money we need to pay him. After that we calculate which of this is already paid and we return the difference.

def __str__(self):
    return self.title

def tag_balance(self):
    return '%s %s' % (self.balance, CURRENCY)

def tag_occupation(self):
    return f'{self.occupation.title}' if self.occupation else 'No data'

def get_card_url(self):
    return reverse('person_card', kwargs={'pk': self.id})

def get_delete_url(self):
    return reverse('person_delete', kwargs={'pk': self.id})

@staticmethod
def filters_data(request, queryset):
    search_name = request.GET.get('search_name', None)
    occup_name = request.GET.getlist('occup_name', None)
    queryset = queryset.filter(title__icontains=search_name) if search_name else queryset
    queryset = queryset.filter(occupation__id__in=occup_name) if occup_name else queryset

    return queryset
And we add this code for the front end again. Now its time for the Payrolls.

First we will add the manager here.

class PayrollInvoiceManager(models.Manager):
    def invoice_per_person(self, instance):
        return super(PayrollInvoiceManager, self).filter(person=instance)

    def not_paid(self):
        return super(PayrollInvoiceManager, self).filter(is_paid=False)
Normal we should create a separate file, but cba. What we need the manager for? We can create our own querysets, which is easier to use them to get the data we want.

Now the Payroll Model.

class Payroll(DefaultOrderModel):
    title = models.CharField(max_length=150, blank=True)
    person = models.ForeignKey(Person, on_delete=models.PROTECT,
                               related_name='person_invoices')
    category = models.CharField(max_length=1, choices=PAYROLL_CHOICES, default='1')
    objects = models.Manager()

    class Meta:
        ordering = ['is_paid', '-date_expired', ]

    def __str__(self):
        return '%s %s' % (self.date_expired, self.person.title)
With the payroll model we can add payments to our employers.The fields now, title we add a note about the payment(optionally), person we see which employer gets the payment, and choice is the type of payment.With ordering, we can handle which order will be show on frontend

The functions now.

def __str__(self):
    return '%s %s' % (self.date_expired, self.person.title)

def save(self, *args, **kwargs):
    self.final_value = self.value
    self.paid_value = self.final_value if self.is_paid else 0
    if self.id:
        self.title = f'Μισθοδοσια {self.id}' if not self.title else self.title
    super(Payroll, self).save(*args, **kwargs)
    self.person.save()
With str we return the desire title we want, and we override the save method, so we can add the paid value if the object is paid and add a default title if the user left it False.

And finally let’s see the signal.

(signals is a django system to call a function when a object is saved, created or deleted, here will use the post_delete)

@receiver(post_delete, sender=Payroll)
def update_person_on_delete(sender, instance, *args, **kwargs):
    person = instance.person
    person.balance -= instance.final_value - instance.paid_value
    person.save()
What we do here? We using the receiver and when a payroll object is deleted, after the deletion we trigger this function to update the person balance.

And finally let’s create another one file called calendar_models.py

from django.db import models
from django.shortcuts import reverse
from .models import Person
from django.conf import settings

from decimal import Decimal


CURRENCY = settings.CURRENCY
SCHEDULE_CHOICES = (
    ('a', 'Normal Time'),
    ('b', 'Extra time')
)
Imports, nothing fancy here.

class PersonSchedule(models.Model):
    date_start = models.DateTimeField(verbose_name='From')
    date_end = models.DateTimeField(verbose_name='Until')
    person = models.ForeignKey(Person, on_delete=models.CASCADE, related_name='schedules')
    hours = models.DecimalField(default=0, decimal_places=2, max_digits=10)
    category = models.CharField(choices=SCHEDULE_CHOICES, default='a', max_length=1)
    cost = models.DecimalField(default=0, decimal_places=2, max_length=20, max_digits=20)

    class Meta:
        ordering = ['-date_start']

    def __str__(self):
        return f'Schedule - {self.id}'

    def save(self, *args, **kwargs):
        diff = self.date_end - self.date_start
        days, seconds = diff.days, diff.seconds
        self.hours = Decimal(days * 24) + Decimal(seconds / 3600)

        if self.category == 'a':
            self.cost = self.hours*self.person.value_per_hour
        else:
            self.cost = self.hours * self.person.extra_per_hour
        print('total cost', self.hours, self.cost)
        super().save(*args, **kwargs)

    def get_delete_url(self):
        return reverse('payroll_bills:delete_schedule', kwargs={'pk': self.id})

    def tag_value(self):
        return f'{self.cost} {CURRENCY}'
This model will help us to track when the employers start and end the shift. Again we override the save method and when a new object is created, we calculate the hours and the cost depends on the type of shift and save it again on cost field.

Now lets create another file the widget.py

from django.forms import DateTimeInput


class XDSoftDateTimePickerInput(DateTimeInput):
    template_name = 'widgets/xdsoft_datetimepicker.html'
This will help us to create a widget for adding date and hour to our browser


Now to use this widget we need our forms.py

from django import forms
from .models import Payroll, Person, Occupation
from .widget import XDSoftDateTimePickerInput
from .calendar_models import PersonSchedule


class BaseForm(forms.Form):

    def __init__(self, *args, **kwargs):
        super().__init__(*args, **kwargs)
        for field_name, field in self.fields.items():
            field.widget.attrs['class'] = 'form-control'
Here we can see the imports and the BaseForm. Will use the BaseForm model for all ours forms we will create have the class form-control which bootstrap understands.

class PersonScheduleForm(BaseForm, forms.ModelForm):
    date_start = forms.DateTimeField(
        input_formats=['%d/%m/%Y %H:%M'],
        widget=XDSoftDateTimePickerInput(attrs={'autocomplete': 'off'}), label='From..'
    )

    date_end = forms.DateTimeField(
        input_formats=['%d/%m/%Y %H:%M'],
        widget=XDSoftDateTimePickerInput(attrs={'autocomplete': 'off'}), label='Until...'
    )
    person = forms.ModelChoiceField(queryset=Person.objects.all(), widget=forms.HiddenInput())

    class Meta:
        model = PersonSchedule
        fields = ['date_start', 'date_end', 'person', 'category']


class PayrollForm(BaseForm, forms.ModelForm):
    date_expired = forms.DateField(required=True, widget=forms.DateInput(attrs={'type': 'date'}), label='Date')

    class Meta:
        model = Payroll
        fields = ['is_paid', 'date_expired', 'person', 'title', 'category', 'value', 'notes', ]


class PersonForm(BaseForm, forms.ModelForm):

    class Meta:
        model = Person
        fields = ['active', 'title', 'phone', 'phone1', 'occupation',
         'value_per_hour', 'extra_per_hour'
                  ]


class PayrollPersonForm(PayrollForm):
    person = forms.ModelChoiceField(queryset=Person.objects.all(), widget=forms.HiddenInput())


class OccupationForm(BaseForm, forms.ModelForm):
    
    class Meta:
        model = Occupation
        fields = ("title",)
All the forms we need is here. A little explain now for PersonScheduleForm

date_start = forms.DateTimeField(
    input_formats=['%d/%m/%Y %H:%M'],
    widget=XDSoftDateTimePickerInput(attrs={'autocomplete': 'off'}), label='From..'
)
We needed way to pass the data from model to frontend, and configure the front end, this line does all the magic.All the others form not need much explain, when we use the HiddenInput, will pass the data from the view and for date_expired, except the form-control we need to pass the date type for the browser again.

4. The views and the templates

I will not cover full explain for the templates files, but only when needed, now let’s see the views.py from frontend folder.

First the imports

from django.shortcuts import reverse, get_object_or_404, redirect, HttpResponseRedirect
from django.contrib import messages
from django.views.generic import  UpdateView, TemplateView
from django.utils.decorators import method_decorator
from django.contrib.admin.views.decorators import staff_member_required
from django.conf import settings

from payroll.models import Payroll, Occupation, Person
from payroll.forms import PayrollForm, PersonForm, OccupationForm, PayrollPersonForm, PersonScheduleForm
from payroll.calendar_models import PersonSchedule


CURRENCY = settings.CURRENCY

The homepage view
Homepage view, we will use always the method_decorator with the staff_member to all views to ensure only staff members have access to the page. Then we add the forms we created before so the user can create a occupation or employer. Finally we will fetch all the data from the models person and occupation with the objects.all()

@method_decorator(staff_member_required, name='dispatch')
class HomepageView(TemplateView):
    template_name = 'homepage.html'

    def get_context_data(self, **kwargs):
        context = super().get_context_data(**kwargs)
        context["create_form"] = PersonForm()
        context['create_occup'] = OccupationForm()
        context['persons'] = Person.objects.all()
        context['occupations'] = Occupation.objects.all()
        return context
Next we create views to validate the data from the forms. What we do? Because on the homepage we have multiply forms weneed to redirect the validation of data with the help of action on the html forms to different views for easy manipulation the post data.

Where is the the action? Inside homepage.html we redirect the forms easy here.

<form method="post" action='{% url "create_person" %}' class="form">
Next.

@staff_member_required
def create_occupation_view(request):
    form = OccupationForm(request.POST or None)
    if form.is_valid():
        form.save()
        messages.success(request, 'New Occupation added!')
    return redirect(reverse('homepage'))


@staff_member_required
def create_person_view(request):
    form = PersonForm(request.POST or None)
    if form.is_valid():
        form.save()
        messages.success(request, 'New Person added!')
    else:
        print('error')
        messages.warning(request, form.errors)
    return redirect(reverse('homepage'))
After the redirection we use this views to get and save the data

And last the update and delete view for occupation.

@method_decorator(staff_member_required, name='dispatch')
class OccupationUpdateView(UpdateView):
    template_name = 'form.html'
    model = Occupation
    form_class = OccupationForm

    def get_context_data(self, **kwargs):
        context = super(OccupationUpdateView, self).get_context_data(**kwargs)
        context['form_title'] = f'Update {self.object}'
        context['back_url'] = reverse('homepage')
        context['delete_url'] = reverse('')
        return context
    
    def form_valid(self, form):
        form.save()
        return self.form_valid(form)


@staff_member_required
def occupation_delete_view(request, pk):
    obj = get_object_or_404(Occupation, id=pk)
    obj.delete()
    return redirect(reverse('homepage'))
Now let’s see the person update view. Here we need to do many things, like update or delete the person, add payment or add schedule.

@method_decorator(staff_member_required, name='dispatch')
class PersonCardView(UpdateView):
    model = Person
    template_name = 'person_view.html'
    form_class = PersonForm

    def get_success_url(self):
        return reverse('person_card', kwargs={'pk': self.object.id})

    def get_context_data(self, **kwargs):
        context = super().get_context_data(**kwargs)
        context['date_filter'] = True
        context['page_title'] = self.object
        context['payment_form'] = PayrollPersonForm(initial={'person': self.object})
        context['calendar_form'] = PersonScheduleForm(initial={'person': self.object})
        context['payments'] = self.object.person_invoices.all()
        context['schedules'] = self.object.schedules.all()
        return context

    def form_valid(self, form):
        form.save()
        return super(PersonCardView, self).form_valid(form)
Let’s analyse one by one.


person Form
First the form to update the data for the person. Because we use class based view, we just add the form to form class and with form_valid, and get_success url we manipulate the post data.

Now the payment form.


Here the user will add the payments. He choose the fields he want and after that with the help of action we redirect it to another view fo validation.

Now let’s explain the time form.


After we pick the from and until fields, we have to choose a category, and depends on category we multiply the time with the hours or extra field from model. Same like before the validation is on another view.

@staff_member_required
def handle_payroll_view(request, pk, type_):
    if type_ == 'delete':
        obj = get_object_or_404(Payroll, id=pk)
        obj.delete()
    elif type_ == 'create':
        form = PayrollForm(request.POST or None)
        if form.is_valid():
            form.save()
            messages.success(request, 'New Payroll Added')
    return HttpResponseRedirect(request.META.get('HTTP_REFERER'))


@staff_member_required
def handle_schedule_view(request, pk, type_):
    if type_ == 'delete':
        obj = get_object_or_404(PersonSchedule, id=pk)
        obj.delete()
    elif type_ == 'create':
        print('create')
        form = PersonScheduleForm(request.POST or None)
        print(form.errors)
        if form.is_valid():
            print('form valid')
            form.save()
            messages.success(request, 'New Payroll Added')
        else:
            print(form.errors)
    return HttpResponseRedirect(request.META.get('HTTP_REFERER'))


def delete_person_view(request, pk):
    person = get_object_or_404(Person, id=pk)
    person.delete()
    return redirect(reverse('homepage'))
