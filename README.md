# help-desk-backend-team-jasur2
# help-desk-backend-team-jasur
core/settings.py faylida DRF va JWT autentifikatsiyasini sozlash

INSTALLED_APPS = [
    ...
    'rest_framework',
    'rest_framework.authtoken',
    'api',
]

REST_FRAMEWORK = {
    'DEFAULT_AUTHENTICATION_CLASSES': (
        'rest_framework_simplejwt.authentication.JWTAuthentication',
    ),
    'DEFAULT_PERMISSION_CLASSES': [
        'rest_framework.permissions.IsAuthenticated',
    ],
}
2. Ma'lumotlar bazasi modellari (api/models.py)
Tickets va replies uchun modellarni yozish
from django.contrib.auth.models import User
from django.db import models

class Ticket(models.Model):
    STATUS_CHOICES = (
        ('open', 'Open'),
        ('in_progress', 'In Progress'),
        ('closed', 'Closed'),
    )
    
    title = models.CharField(max_length=255)
    description = models.TextField()
    status = models.CharField(max_length=20, choices=STATUS_CHOICES, default='open')
    created_by = models.ForeignKey(User, on_delete=models.CASCADE, related_name='tickets')
    created_at = models.DateTimeField(auto_now_add=True)

    def __str__(self):
        return self.title

class Reply(models.Model):
    ticket = models.ForeignKey(Ticket, on_delete=models.CASCADE, related_name='replies')
    author = models.ForeignKey(User, on_delete=models.CASCADE)
    body = models.TextField()
    created_at = models.DateTimeField(auto_now_add=True)

    def __str__(self):
        return f"Reply by {self.author.username} on {self.ticket.title}"


3. Serialayzerlar (api/serializers.py)
Ma'lumotlarni validatsiya qilish va JSON formatiga o'tkazish uchun

from django.contrib.auth.models import User
from rest_framework import serializers
from .models import Ticket, Reply

class RegisterSerializer(serializers.ModelSerializer):
    password = serializers.CharField(write_only=True)

    class Meta:
        model = User
        fields = ('username', 'password', 'email')

    def create(self, validated_data):
        user = User.objects.create_user(
            username=validated_data['username'],
            email=validated_data.get('email', ''),
            password=validated_data['password']
        )
        return user

class TicketSerializer(serializers.ModelSerializer):
    created_by = serializers.ReadOnlyField(source='created_by.username')

    class Meta:
        model = Ticket
        fields = ('id', 'title', 'description', 'status', 'created_by', 'created_at')

class TicketCreateSerializer(serializers.ModelSerializer):
    class Meta:
        model = Ticket
        fields = ('title', 'description')

class TicketStatusSerializer(serializers.ModelSerializer):
    class Meta:
        model = Ticket
        fields = ('status',)

    def validate_status(self, value):
        valid_statuses = ['open', 'in_progress', 'closed']
        if value not in valid_statuses:
            raise serializers.ValidationError("Noto'g'ri status kiritildi.")
        return value

class ReplySerializer(serializers.ModelSerializer):
    author = serializers.ReadOnlyField(source='author.username')

    class Meta:
        model = Reply
        fields = ('id', 'ticket', 'body', 'author', 'created_at')

class ReplyCreateSerializer(serializers.ModelSerializer):
    class Meta:
        model = Reply
        fields = ('ticket', 'body')


4. API Endpoyntlari va Vyuws (api/views.py)
Talab qilingan barcha amallarni bajaruvchi viewlar

from django.contrib.auth.models import User
from rest_framework import generics, permissions, status
from rest_framework.response import Response
from rest_framework_simplejwt.tokens import RefreshToken
from .models import Ticket, Reply
from .serializers import (
    RegisterSerializer, TicketSerializer, TicketCreateSerializer,
    TicketStatusSerializer, ReplySerializer, ReplyCreateSerializer
)

# Auth: Register
class RegisterView(generics.CreateAPIView):
    queryset = User.objects.all()
    permission_classes = [permissions.AllowAny]
    serializer_class = RegisterSerializer

# Auth: Login (Token va username qaytaradi)
from rest_framework.views import APIView
from django.contrib.auth import authenticate

class LoginView(APIView):
    permission_classes = [permissions.AllowAny]

    def post(self, request):
        username = request.data.get('username')
        password = request.data.get('password')
        user = authenticate(username=username, password=password)
        
        if user is not None:
            refresh = RefreshToken.for_user(user)
            return Response({
                'token': str(refresh.access_token),
                'username': user.username
            }, status=status.HTTP_200_OK)
        return Response({'error': 'Login yoki parol noto‘g‘ri'}, status=status.HTTP_400_BAD_REQUEST)

# Tickets: List va Create
class TicketListCreateView(generics.ListCreateAPIView):
    queryset = Ticket.objects.all()
    
    def get_serializer_class(self):
        if self.request.method == 'POST':
            return TicketCreateSerializer
        return TicketSerializer

    def perform_create(self, serializer):
        serializer.save(created_by=self.request.user)

# Tickets: Get (Detail)
class TicketDetailView(generics.RetrieveAPIView):
    queryset = Ticket.objects.all()
    serializer_class = TicketSerializer
    lookup_field = 'pk'

# Tickets: Set Status
class TicketSetStatusView(generics.UpdateAPIView):
    queryset = Ticket.objects.all()
    serializer_class = TicketStatusSerializer
    lookup_field = 'pk'

    def patch(self, request, *args, **kwargs):
        return self.partial_update(request, *args, **kwargs)

# Replies: Create
class ReplyCreateView(generics.CreateAPIView):
    queryset = Reply.objects.all()
    serializer_class = ReplyCreateSerializer

    def perform_create(self, serializer):
        serializer.save(author=self.request.user)


5. URL manzillari (api/urls.py va core/urls.py)

from django.urls import path
from .views import (
    RegisterView, LoginView, TicketListCreateView,
    TicketDetailView, TicketSetStatusView, ReplyCreateView
)

urlpatterns = [
    path('auth/register', RegisterView.as_view(), name='auth-register'),
    path('auth/login', LoginView.as_view(), name='auth-login'),
    path('tickets', TicketListCreateView.as_view(), name='ticket-list-create'),
    path('tickets/<int:pk>', TicketDetailView.as_view(), name='ticket-detail'),
    path('tickets/<int:pk>/set-status', TicketSetStatusView.as_view(), name='ticket-set-status'),
    path('replies', ReplyCreateView.as_view(), name='reply-create'),
]

from django.contrib import admin
from django.urls import path, include

urlpatterns = [
    path('admin/', admin.site.urls),
    path('api/', include('api.urls')),
]

6. Loyihani ishga tushirish qadamlari (README.md)

# Django Backend API - Tickets & Auth

## Talablar
- Python 3.10+
- Pip

## O'rnatish qadamlari

1. Repozitoriyani klon qiling va papkaga o'ting:
   ```bash
   git clone <repo-url>
   cd <repo-folder>

python -m venv venv
# Windows uchun:
venv\Scripts\activate
# Mac/Linux uchun:
source venv/bin/activate

Kerakli kutubxonalarni o'rnatim
pip install django djangorestframework djangorestframework-simplejwt
python manage.py makemigrations
python manage.py migrate
python manage.py runserver
 API quyidagi manzilda ishlaydihttp://127.0.0.1:5000/
from django.db import models
from django.contrib.auth.models import User

class Ticket(models.Model):
    STATUS_CHOICES = (
        ('open', 'Open'),
        ('in_progress', 'In Progress'),
        ('closed', 'Closed'),
    )

    title = models.CharField(max_length=255)
    description = models.TextField()
    status = models.CharField(max_length=20, choices=STATUS_CHOICES, default='open')
    created_by = models.ForeignKey(User, on_delete=models.CASCADE, related_name='tickets')
    created_at = models.DateTimeField(auto_now_add=True)

    def __str__(self):
        return self.title


class Reply(models.Model):
    ticket = models.ForeignKey(Ticket, on_delete=models.CASCADE, related_name='replies')
    author = models.ForeignKey(User, on_delete=models.CASCADE)
    body = models.TextField()
    created_at = models.DateTimeField(auto_now_add=True)

    def __str__(self):
        return f"Reply to {self.ticket.id} by {self.author.username}"
        from django.contrib.auth.models import User
from rest_framework import serializers
from .models import Ticket, Reply

class RegisterSerializer(serializers.ModelSerializer):
    password = serializers.CharField(write_only=True)

    class Meta:
        model = User
        fields = ('username', 'password', 'email')

    def create(self, validated_data):
        user = User.objects.create_user(
            username=validated_data['username'],
            email=validated_data.get('email', ''),
            password=validated_data['password']
        )
        return user


class TicketSerializer(serializers.ModelSerializer):
    created_by = serializers.ReadOnlyField(source='created_by.username')

    class Meta:
        model = Ticket
        fields = ('id', 'title', 'description', 'status', 'created_by', 'created_at')


class TicketCreateSerializer(serializers.ModelSerializer):
    class Meta:
        model = Ticket
        fields = ('title', 'description')


class TicketStatusSerializer(serializers.ModelSerializer):
    class Meta:
        model = Ticket
        fields = ('status',)

    def validate_status(self, value):
        if value not in ['open', 'in_progress', 'closed']:
            raise serializers.ValidationError("Noto'g'ri status.")
        return value


class ReplySerializer(serializers.ModelSerializer):
    author = serializers.ReadOnlyField(source='author.username')

    class Meta:
        model = Reply
        fields = ('id', 'ticket', 'body', 'author', 'created_at')


class ReplyCreateSerializer(serializers.ModelSerializer):
    class Meta:
        model = Reply
        fields = ('ticket', 'body')
        from django.contrib.auth.models import User
from django.contrib.auth import authenticate
from rest_framework import generics, permissions, status
from rest_framework.views import APIView
from rest_framework.response import Response
from rest_framework_simplejwt.tokens import RefreshToken

from .models import Ticket, Reply
from .serializers import (
    RegisterSerializer, TicketSerializer, TicketCreateSerializer,
    TicketStatusSerializer, ReplySerializer, ReplyCreateSerializer
)

class RegisterView(generics.CreateAPIView):
    queryset = User.objects.all()
    permission_classes = [permissions.AllowAny]
    serializer_class = RegisterSerializer


class LoginView(APIView):
    permission_classes = [permissions.AllowAny]

    def post(self, request):
        username = request.data.get('username')
        password = request.data.get('password')
        user = authenticate(username=username, password=password)

        if user is not None:
            refresh = RefreshToken.for_user(user)
            return Response({
                'token': str(refresh.access_token),
                'username': user.username
            }, status=status.HTTP_200_OK)
        return Response({'error': 'Xato maʼlumot kiritildi'}, status=status.HTTP_400_BAD_REQUEST)


class TicketListCreateView(generics.ListCreateAPIView):
    queryset = Ticket.objects.all()

    def get_serializer_class(self):
        if self.request.method == 'POST':
            return TicketCreateSerializer
        return TicketSerializer

    def perform_create(self, serializer):
        serializer.save(created_by=self.request.user)


class TicketDetailView(generics.RetrieveAPIView):
    queryset = Ticket.objects.all()
    serializer_class = TicketSerializer
    lookup_field = 'pk'


class TicketSetStatusView(generics.UpdateAPIView):
    queryset = Ticket.objects.all()
    serializer_class = TicketStatusSerializer
    lookup_field = 'pk'

    def patch(self, request, *args, **kwargs):
        return self.partial_update(request, *args, **kwargs)


class ReplyCreateView(generics.CreateAPIView):
    queryset = Reply.objects.all()
    serializer_class = ReplyCreateSerializer

    def perform_create(self, serializer):
        serializer.save(author=self.request.user)

        from django.urls import path
from .views import (
    RegisterView, LoginView, TicketListCreateView,
    TicketDetailView, TicketSetStatusView, ReplyCreateView
)

urlpatterns = [
    path('auth/register', RegisterView.as_view(), name='auth-register'),
    path('auth/login', LoginView.as_view(), name='auth-login'),
    path('tickets', TicketListCreateView.as_view(), name='ticket-list-create'),
    path('tickets/<int:pk>', TicketDetailView.as_view(), name='ticket-detail'),
    path('tickets/<int:pk>/set-status', TicketSetStatusView.as_view(), name='ticket-set-status'),
    path('replies', ReplyCreateView.as_view(), name='reply-create'),
]
# Backend API Project

## O'rnatish va Ishga Tushirish

1. Virtual muhitni faollashtiring:
```bash
python -m venv venv
source venv/bin/activate  # Windows uchun: venv\Scripts\activate
