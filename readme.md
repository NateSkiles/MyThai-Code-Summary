# Python Live Project

### Key
* [Introduction](https://github.com/NateSkiles/Python-Live-Project-Code-Summary#introduction)
* [CRUD Functionality](https://github.com/NateSkiles/Python-Live-Project-Code-Summary#crud-functionality)
* [API](https://github.com/NateSkiles/Python-Live-Project-Code-Summary#api)
* [Front-End](https://github.com/NateSkiles/Python-Live-Project-Code-Summary#front-end-development)
* [Skills Acquired](https://github.com/NateSkiles/Python-Live-Project-Code-Summary#skills-acquired)


## Introduction
The task of this project was to create an app in the Django framework that would help the user keep track of a collection of items. The app I created is used to store the user's favorite Thai food takeout. The user can add a restaurant they ordered from and then add a dish with a rating and description to the app. This allows you to search or sort the SQLite database to find specific dishes from multiple restaurants and compare the ratings they gave to those dishes.

Note: Because this project was done in close collaboration with my team and fellow students, I may not post the project in its entirety. Instead, I have included my HTML templates, CSS & images, as well as this code summary to document my work and experiences during the two-week live project at the Tech Academy.


## CRUD Functionality

### Create
The first thing I had to do was create models for my app. These models are used to add objects to the database. For this project, I used two models; one for the *Dishes*  and one for *Restaurants*. To do this, I used Django's Model class and defined my models' attributes in the *models.py* file of the app: 

``` python
class Restaurant(models.Model):
    name = models.CharField(max_length=40)
    phone = models.CharField(max_length=13, default='(000)000-0000')
    address = models.CharField(max_length=95)
    city = models.CharField(max_length=15, default='Portland')
    rating = models.IntegerField(
        validators=[MaxValueValidator(10), MinValueValidator(0)]  # Check to make sure rating in range.
    )
    
    objects = models.Manager()
	
    def __str__(self):
        return self.name
```
``` python
class Dish(models.Model):
    dishName = models.CharField(max_length=40)
    dishType = models.CharField(max_length=10, choices=DISH_TYPES)
    restaurant = models.ForeignKey(Restaurant, on_delete=models.CASCADE)
    rating = models.IntegerField(
        validators=[MaxValueValidator(10), MinValueValidator(0)]  # Check to make sure rating in range.
    )
    description = models.CharField(max_length=255)
	
    objects = models.Manager()
	
    def __str__(self):
        return self.dishName
```
Next, to get information from the user, I used a built-in model to display a form to the user. Which, when returned with valid data, will create a new object in the database. Done in the forms.py project file.

``` python
class RestaurantForm(ModelForm):
    class Meta:
        model = Restaurant
        fields = '__all__'


class DishForm(ModelForm):
    description = forms.CharField(widget=forms.Textarea)

    class Meta:
        model = Dish
        fields = '__all__'
```
To display the form to the user, I created a template that uses a form with a request to POST new objects to the database.

Note: For the upcoming examples, there are templates and functions for both Restaurants and Dishes. I have only included one to prevent excessive redundancy. 

``` Jinja
{% extends "MyThai/MyThai_base.html" %}

{% block title %}MyThai! | Add Dish{% endblock %}

{% block content %}

    <div id="dish-form">
        <form method="POST">
            {% csrf_token %}
            {{form.as_p}}
            <button id="new-dish-btn" type="submit" name="Save_Dish">Save Dish</button>
        </form>
        <br>
    </div>

{% endblock %}
```
This template extends the MyThai_base.html file, which is the base template for this project. 

Finally, to render the pages that add restaurants and dishes, I created a view that processes and brings all these things together. 

``` python
def new_dish(request):
    form = DishForm(data=request.POST or None)
    if request.method == 'POST':
        if form.is_valid():
            form.save()
            return redirect('MyThai_my_restaurants')
        else:
            content = {'form': form}
            return render(request, 'MyThai/MyThai_add_dish.html', content)
    content = {'form': form}
    return render(request, 'MyThai/MyThai_add_dish.html', content)
```
This block of code creates an object called form from the data the user's form POSTed. If the form is valid, it is saved to the database.

![Add Dish](https://github.com/NateSkiles/Python-Live-Project-Code-Summary/blob/main/images/add_dish.gif)

### Read
Once the user can add objects to the database, they need to be able to view this data. I did this by using DTL to populate an HTML table with the information in the database.

*views.py*

``` python
def my_restaurants_view(request):
    # Store objects from DB in object as dict.
    dish_list = Dish.objects.all()
    # GET search & sort data for table
    get_dish_query = request.GET.get('get_dish')
    my_sort = request.GET.get('dishes')

    if is_valid_query(get_dish_query):  # If 'is valid' = True, filter the query-set
        dish_list = dish_list.filter(
            # Turn filters into objects to pass to filter()
            Q(dishName__icontains=get_dish_query) |  # Search by dish name & restaurant name
            Q(restaurant__name__icontains=get_dish_query)).distinct()  # Returning only distinct entries

    dish_list = my_sorted(dish_list, my_sort)  # Sort qs
    paginator = Paginator(dish_list, 10)  # Create paginator object with 10 restaurants per page
    page = request.GET.get('page')  # Store paginator object with current page
    dishes = paginator.get_page(page)

    context = {'dishes': dishes}
    return render(request, 'MyThai/MyThai_my_dishes.html', context)
```
In this block of code, I query the database to return all of its objects using Django's Object Relation Mapping. ORM lets you interact with a database (SQLite in this case) as you would with SQL, but written in python. I use Django Q objects to create filter objects, allowing me to filter the query-set by Dish Name or Restaurant Name, instead of only one or the other. 

I also used Paginator through this project to create paged objects for neater display in the templates. I also call a helper function *my_sorted()*, which is described [here](https://github.com/NateSkiles/Python-Live-Project-Code-Summary#search-method-ofsearchform).

![sort](https://github.com/NateSkiles/Python-Live-Project-Code-Summary/blob/main/images/My%20Thai!%20_%20Home.gif)

``` python
def my_sorted(dish_list, my_sort):
    if my_sort == 'rating':  # If sorted by rating, reverse so highest goes at the top.
        asc = True
    elif my_sort is None:  # If my_sort is None, sort by dish name
        my_sort = 'dishName'
        asc = False
    else:
        asc = False
    dish_list = sorted(dish_list, key=attrgetter(my_sort), reverse=asc)  # Sort dict by model attribute = 'my_sort'
    return dish_list
```
This function sorts the query-set based on the parameters passed in the GET request. This function had me stuck for a bit, I couldn't get my parameter *'my_sort'* to be a valid key in the *sorted()* function. After searching the Django documentation I figured out I needed to turn *my_sort* parameter into a callable object, and the best way to do that is with the python function *attrgetter()*. Attrgetter returns an attribute of an object as a callable object. 

*HTML template to display objects*:

``` Jinja
{% for dish in dishes %}
	<tr>
		<td><span><a href="{% url 'MyThai_details' dish.id %}">{{ dish.dishName|capfirst }}</a></span></td>
		<td>{{ dish.dishType|capfirst }}</td>
		<td>{{ dish.description|capfirst }}</td>
		<td><span><a href="{% url 'MyThai_rest_details' dish.restaurant_id %}">{{ dish.restaurant|capfirst }}</a></span></td>
		<td>{{ dish.rating }}</td>
	</tr>
{% endfor %}
```          
![My Dishes](https://github.com/NateSkiles/Python-Live-Project-Code-Summary/blob/0f9ffa67d5aeb1f8c2e1831c025d1c08cb3e6818/images/MyThai%20-%20My%20Dishes.png)

### Update & Delete
From the table created in the *my\_restaurants_view()* template, I made all the names for the dishes and restaurants hyperlinks. The links take the user to a details page that displays all the information on that object allowing the user to edit or delete that object. 

The view for the details page(s):

*views.py*

``` python
def details(request, pk):
    pk = int(pk)
    # Get object with pk
    dish = get_object_or_404(Dish, pk=pk) # Get dish object or return 404
    context = {'dish': dish}
    return render(request, 'MyThai/MyThai_details.html', context)
```

Once on the views page, the user edits the object through this view:

``` python
def dish_edit(request, pk):
    pk = int(pk)
    item = get_object_or_404(Dish, pk=pk)
    form = DishForm(data=request.POST or None, instance=item)
    if request.method == 'POST':
        save_edit(form)  # If form is valid, save and redirect to all dishes page
        return redirect('MyThai_my_restaurants')
    else:
        return render(request, 'MyThai/MyThai_dish_edit.html', {'form': form})
```
This view is passed the objects Primary Key (*PK*) and the request. These used to populate our DishForm with the information stored in the database under that primary key. When the user clicks the submit button in the template, a POST request is made to save the form (object). The function *save_edit()* check the validity of the form and returns any errors.

We can delete an object in the database much like we edited one:

``` python
def dish_delete(request, pk):
    pk = int(pk)
    item = get_object_or_404(Dish, pk=pk)
    if request.method == 'POST':
        item.delete()
        return redirect('MyThai_my_restaurants')
    context = {"item": item}
    return render(request, "MyThai/MyThai_delete.html", context)
```

![Delete](https://github.com/NateSkiles/Python-Live-Project-Code-Summary/blob/main/images/delete.gif)

However, instead of saving when a POST request is made the user is redirected to a confirmation page assuring they want to delete the item selected from the database.

*Back to [Top](https://github.com/NateSkiles/Python-Live-Project-Code-Summary#python-live-project)*

## API
I also implemented a search feature for using the Yelp Fusion API to return a JSON response based on the user's search. To accomplish, this I first created another form to get a search term from the user.

*forms.py*

``` python
class SearchForm(forms.Form):
    search_term = forms.CharField(max_length=100)
```
Here is the views function to render the search form and then make a GET request when the SearchForm is submitted. 

``` python
def restaurant_search(request):
    search_result = {}		
	# If there is a GET request submitted
    if 'term' in request.GET:
    	# Save SearchForm GET request as form
        form = SearchForm(request.GET)
        if form.is_valid():
        	# If form is valid, run method .search()
            search_result = form.search()
    else:
        form = SearchForm()

    context = {'form': form, 'search_result': search_result}

    return render(request, 'MyThai/MyThai_api_search.html', context)
```
This block of code renders the SearchForm as a prompt for the user to search for a Thai food restaurant in Portland, OR. Once the form submits via a GET request, if valid, the SearchForm will call the method *.search()*, store the results, and then pass the search results back to the page.


#### .search() Method ofSearchForm

Once the method is called, the search term is entered into the form is cleaned, and then set as the value to the key  'term' in a dictionary of the URL parameter that we will use later to make a request from the API.

``` python
class SearchForm(forms.Form):
    search_term = forms.CharField(max_length=100)

    def search(self):
   	API_KEY = 'xxxxxxxxxxx'
        API_HOST = 'https://api.yelp.com'
        SEARCH_PATH = '/v3/businesses/{}'.format('search')
        SEARCH_LIMIT = 5
        SEARCH_CATEGORY = 'Thai'
        SEARCH_LOCATION = 'Portland, OR'
	
    	term = self.cleaned_data['search_term']

        url_params = {
            'term': term.replace(' ', '+'),  # Remove spaces from params
            'location': SEARCH_LOCATION.replace(' ', '+'),
            'limit': SEARCH_LIMIT,
            'categories': SEARCH_CATEGORY
        }
        ...
```
Next is to define the URL variable and pass the API key to the headers used in the API request. Now everything is ready to be passed into a GET request that will hopefully contain the response to the request.

``` python
		...
		# Combine host and path to make url
        url = '{}{}'.format(API_HOST, quote(SEARCH_PATH.encode('utf8')))
        
        # Authorization header for API
        headers = {
            'Authorization': 'Bearer %s' % API_KEY,
        }
        
        # Construct API get request
        response = requests.request('GET', url, headers=headers, params=url_params)
        ...
```
Check the response code from the API: 200 means the request was successful, 404 that there were no results found, and any other code will return an error to the user. If the status code is 200, return a JSON object of the response.

``` python
		...
        result = {}

        # If status_code is successful
        if response.status_code == 200:
        # Return dict response as json
            result = response.json()
            result['success'] = True
            
        else:
            result['success'] = False
            if response.status_code == 404:
                result['message'] = 'No entry found for "%s' % term
            else:
                result['message'] = 'The Yelp Fusion API is not available at the movement, sorry.'
                
        return result
```

From there the results are passed via the variable context back to the view to be rendered to the user.
![API Results](https://github.com/NateSkiles/Python-Live-Project-Code-Summary/blob/main/images/MyThai!%20_%20Search.gif)

*Back to [Top](https://github.com/NateSkiles/Python-Live-Project-Code-Summary#python-live-project)*

## Front-End Development
One of the things I would like to come back to is redoing the Front-End with Bootstrap. I wrote out the CSS to save time getting Bootstrap to work for this project. I have included a few pictures to illustrate my app and its functionality outlined above.

### Homepage
![Homepage](https://github.com/NateSkiles/Python-Live-Project-Code-Summary/blob/0f9ffa67d5aeb1f8c2e1831c025d1c08cb3e6818/images/MyThai%20-%20Home.png)


## Technologies Used
The app is created in the Django Framework version 2.2 and was written with Python, HTML, CSS, & DTL (Django-Template Language). 

## Skills Acquired
I gained many skills during this live project, the one I would like to discuss first is confidence. While there are always things to be improved on, the confidence gained from starting with nothing to having a fully functioning app has given me confidence in my ability to problem solve as a developer. I still have a bit to learn, but this project shows the ability to stick with something even when it looks like you've hit a dead-end. Learning how to get yourself out of those situations through research and asking the right questions will always be a skill I take away from any project I complete.

A takeaway from the project and working as a small piece of a larger project is how vital version control is to the organization of a project. When there’s even one developer working on a project, things can get messy quickly, so being comfortable managing your branches and working around other developer’s branches is an integral part of working as a team. 

Finally, as for technical skills, I gained a lot of knowledge on Python/Django and, more specifically, the logic behind the Model-View-Controller (MVC) framework pattern. Django uses a Template, not a controller, and would be considered a Model-View-Template (MVT). I still gained the skills to understand the patterns behind the framework and how its different parts interact. This gave me the tools to more effectively familiarize myself with new frameworks and technologies down the road.

*Back to [Top](https://github.com/NateSkiles/Python-Live-Project-Code-Summary#python-live-project)*
