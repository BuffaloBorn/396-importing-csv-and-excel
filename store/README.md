In episode 362 we showed how to export database records to a CSV or Excel file. Since then there have been a number of requests for an episode showing how to import records from these types of files so that’s what we’ll cover in this episode.

We’ll do this by adding a form to the bottom of this page that will allow the user to upload a file containing records. When the form is submitted the file will be parsed and the records added to the database. We’ll add the form at the bottom of the view template

_/app/views/products/index.html.erb_
```ruby
<h2>Import Products</h2>
<%= form_tag import_products_path, multipart: true do %>
  <%= file_field_tag :file %>
  <%= submit_tag "Import" %>
<% end %>
```
We don’t have an object to handle the importing right now so we’ve used form_tag instead of form_for. The form will be submitted to a new ```import``` action on the ```ProductsController``` and note that we’ve set the multipart option so that the form can handle file uploads. We’ll need to set up the new path in the routes file so we’ll do that now.

_/config/routes.rb_
```ruby
Store::Application.routes.draw do
  resources :products do
    collection { post :import }
  end
  root to: 'products#index'
end
```
Now in the ```ProductsController``` we’ll add the ```import``` action. This should take the uploaded file then ```import``` its data into our database. The file will be uploaded into a file parameter and Rails will store the uploaded file temporarily in the file system while it’s processed. This means that we don’t have to use CarrierWave or Paperclip to work with uploaded files. In this action we’ll pass the uploaded file to a new ```import``` method on the Product model then redirect back to the home page.
