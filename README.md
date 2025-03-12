# DRF-Email-Verification
DRF Registration->Verification->Token

settings.py:
```
EMAIL_BACKEND = "django.core.mail.backends.console.EmailBackend"
```

models.py:
```
class User(AbstractUser):
    def upload_to(self, filename):
        return filename
    phone_number = models.CharField(
        max_length=255,
        unique=True,
        validators=[phone_validator],
    )
    is_email_confirmed = models.BooleanField(default=False)

    USERNAME_FIELD = "phone_number"

    # REQUIRED_FIELDS = ["first_name"]
    REQUIRED_FIELDS = ["username"]

    def __str__(self):
        return self.phone_number
    
class EmailConfirmationToken(models.Model):
    created_at = models.DateTimeField(auto_now_add=True)
    user = models.ForeignKey(User, on_delete=models.CASCADE)
```

views.py:
```
from django.shortcuts import render
from rest_framework import status
from rest_framework.generics import CreateAPIView
from rest_framework.response import Response
from rest_framework.views import APIView
from rest_framework_simplejwt.authentication import JWTAuthentication
from rest_framework_simplejwt.tokens import AccessToken, RefreshToken

from .models import User, EmailConfirmationToken
from .serializers import CustomTokenObtainPairSerializer, RegisterSerializer
from .utils import send_confirmation_email

class UserInformationAPIView(APIView):
    authentication_classes = [JWTAuthentication]

    def get(self, request):
        try:
            user: User = request.user
            payload = {
                'email': user.email, 
                'is_email_confirmed': user.is_email_confirmed
            }
            return Response(data=payload, status=200)
        except Exception as e:
            return Response({"error": str(e)}, status=401)
    
class SendEmailConfirmationTokenAPIView(APIView):
    authentication_classes = [JWTAuthentication]

    def post(self, request, format=None):
        try:
            user: User = request.user
            token = EmailConfirmationToken.objects.create(user=user)
            send_confirmation_email(email=user.email, token_id=token.pk, user_id=user.pk)
            return Response(data=None, status=201)
        except Exception as e:
            return Response({'error': str(e)}, status=401)

def confirm_email_view(request):
    token_id = request.GET.get('token_id', None)
    user_id = request.GET.get('user_id', None)
    try:
        token = EmailConfirmationToken.objects.get(pk=token_id)
        user = token.user
        user.is_email_confirmed = True
        user.save()
        data = {
            'is_email_confirmed': True
        }
        return render(request, template_name='accounts/confirm_email_view.html', context=data)
    except EmailConfirmationToken.DoesNotExist:
        data = {
            'is_email_confirmed': False
        }
        return render(request, template_name='accounts/confirm_email_view.html', context=data)

```

utils.py:
```

from django.core.mail import send_mail
from django.template.loader import get_template

def send_confirmation_email(email, token_id, user_id):
    data = {
        "token_id":str(token_id),
        "user_id":str(user_id)
    }
    message = get_template('accounts/confirmation_email.txt').render(data)
    send_mail(
        subject='Please confirm email',
        message=message,
        from_email='admin@example.com',
        recipient_list=[email],
        fail_silently=True
    )
    # print('Email confirmation was sent')
```

create file under `accounts/templates/accounts/confirmation_email.txt`:
```

Please confirm your email address by going to the following link:

http://localhost:8000/accounts/confirm-email?token_id={{token_id}}&user_id={{user_id}}
```
create template under `aacounts/templates/accounts/confirm_email_view.html`:
```

{% block main %}

{% if is_email_confirmed == True %}
<div class="notification is-success">
    <p>Your Email is confirmed</p>
</div>
{% else %}
<div class="notification is-danger">
    <p>Your confirmation token is expired.Please request new one</p>
</div>
{% endif %}

{% endblock main %}
```

urls.py:
```
from channels.routing import URLRouter
from django.urls import path
from rest_framework.routers import DefaultRouter

from .consumers import *
from .views import Home, RegisterView, SendEmailConfirmationTokenAPIView, UserInformationAPIView, confirm_email_view

router = DefaultRouter()

app_name = 'users'

urlpatterns = router.urls + [
    path("api/home/", Home.as_view(), name="home"),
    path("api/register/", RegisterView.as_view(), name="register_api_view"),
    path("api/userinfo/", UserInformationAPIView.as_view(), name="user_information_api_view"),
    path("api/send-confirmation-email/", SendEmailConfirmationTokenAPIView.as_view(), name="send_email_confirmation_api_view"),
    path("accounts/confirm-email/", confirm_email_view, name="confirm_email_view"),
]

auth_router = URLRouter([path("test/", TestConsumer.as_asgi())])

```

create `test_views.py` under `aacounts/tests//test_views.py`:
- remember, create `__init__.py`
- RUN: ` python manage.py test accounts.tests.test_views`
```

from django.urls import reverse
from django.contrib.auth import get_user_model
from rest_framework.test import APITestCase
from accounts.models import EmailConfirmationToken

User = get_user_model()

class UsersAPIViewsTests(APITestCase):
    def test_user_information_api_view_requires_authentication(self):
        url = reverse('users:user_information_api_view')
        response = self.client.get(url)
        self.assertEquals(response.status_code, 401)

    def test_send_email_confirmation_api_view_requires_authentication(self):
        url = reverse('users:send_email_confirmation_api_view')
        response = self.client.post(url)
        self.assertEquals(response.status_code, 401)

    def test_send_email_confirmation_api_creates_token(self):
        user = User.objects.create_user(first_name = "new",last_name = "user2",username = "newuser2",password = "admin123j",email = "newuser2@example.com",phone_number = "911223346")
        url = reverse('users:send_email_confirmation_api_view')
        self.client.force_authenticate(user=user)
        response = self.client.post(url)
        self.assertEquals(response.status_code, 201)
        token = EmailConfirmationToken.objects.filter(user=user).first()
        self.assertIsNotNone(token)
```
