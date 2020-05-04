## Setup

1. Generate new rails projects `rails new musicial-market -d postgresql`

2. Create database `rails db:create`
3. Connect database to dbeaver

4. Create homepage as root
   1. `get "/", to: "pages#home", as: "root"`

## Users with Devise

1. Add devise gem `bundle add devise`
2. Generate devise `rails generate devise:install`, and follow instructions to complete setup:
3. Generate devise user model `rails generate devise User`
4. Migrate db `rails db:migrate`
5. Add signup/login, logout links to `application.html.erb`

```html
    <% if user_signed_in? %>
      <%= link_to "logout", destroy_user_session_path, method: :delete %>
    <% else %>
      <%= link_to "login / signup", new_user_session_path, method: :get %>
    <% end %>
```

## Listings CRUD

### Generate Model

1. Generate migration file `rails g model Listing user:references title:string description:string price:integer`
2. Migrate db `rails db:migrate`
3. Update user model to establish relationship with Listing `has_many :listings, dependent: :destroy`
4. Validate required fields in listing model `validates :title, :description, :price, presence: true`
5. Specify that a picture should be attached to the listing `has_one_attached :picture`

### Build listings home page

1. Add route / controller / view

```ruby
get "/listings", to: "listings#index", as: "listings"

###

before_action :authenticate_user!
def index
    @listings = Listing.all
end
```

```html
<h1>Musician Market</h1>

<% for listing in @listings %>
    <%= listings.title %> - <%= number_to_currency(listings.price) %>
<% end %>
```

### Attach s3 for image storage
1.  Install active storage `rails active_storage:install`
    1.  `rails db:migrate`
2.  Add s3 gem `bundle add aws-sdk-s3`
3.  Create s3 bucket
4.  Add s3 bucket info to `config/storage.yml`
5.  Modify active storage service to `:amazon` in `config/development.rb`
6.  Add s3 IAM keys to config file `EDITOR="code --wait" rails credentials:edit`

```
aws:
  access_key_id: 
  secret_saccess_key: 
```

### Create Listing

1. Add *new* route / controller action / view

```ruby
get "/listings/new", to: "listings#new", as: "new_listing"

def new
    @listing = Listing.new
end
```
```html
<%= form_with model: @listing, local: true do |form| %>
    <div>
        <%= form.label :title %>
        <%= form.text_field :title, value: @listing.title %>
    </div>
    <div>
        <%= form.label :description %>
        <%= form.text_area :description, value: @listing.description %>
    </div>
    <div>
        <%= form.label :price %>
        <%= form.number_field :price, min: 0, value: @listing.price %>
    </div>
    <div>
        <%= form.label :picture %>
        <%= form.file_field :picture, accept: "image/png,image/gif,image/jpeg,image/jpg", value: @listing.picture %>
    </div>
    <div>
        <%= form.submit %>
    </div>
<% end %>
```

1. Add *create* route / controller action

```ruby
post "/listings", to: "listings#create"

def create
    current_user.listings.create(listing_params)
    redirect_to listings_path
end

private
def listing_params
    params.require(:listing).permit(:title, :description, :price, :picture)
end
```

2. Add error handling
```ruby
def create
    @listing = current_user.listings.create(listing_params)
    if @listing.errors.any?
        render "new"
    else 
        redirect_to listings_path
    end
end
```
```html
<% if @listing.errors.any? %>
    <div class="errors">
        <% @listing.errors.full_messages.each do |error| %>
            <p><%= error %></p>
        <% end %>
    </div>
<% end %>
```

### Show Listing

```ruby
  get "/listings/:id", to: "listings#show", as: "listing"
```
```html
<h1><%= @listing.title %></h1>
<%= image_tag  @listing.picture, style: "width: 200px;" %>
<h2><%= number_to_currency(@listing.price) %></h2>
<p>Seller email: <%= @listing.user.email %></p>
<% if current_user.id == @listing.user_id%>
    <%= link_to "edit", edit_listing_path(@listing.id)%>
<% end %>
<p><%= @listing.description %></p>
```

### Edit Listing

1. Add *edit* route / controller action / view

```ruby
put "/listings/:id", to: "listings#update"
patch "/listings/:id", to: "listings#update"
get "/listings/:id/edit", to: "listings#edit", as: "edit_listing"

@listing = current_user.listings.find_by_id(params[:id])
if @listing
    @listing.update(listing_params)
    rerender_error('edit')
else 
    redirect_to listings_path
end   

# <%= render "form", listing: @listing %>
```

### Delete Listing
```ruby
delete "/listings/:id", to: "listings#destroy"

def destroy
    Listing.find(params[:id]).destroy
    redirect_to listings_path
end
```
```html
<%= form_with model: @listing, local: true, method: "delete" do |form| %>
    <%= submit_tag "delete"%>
<% end %>
```

### Donations

1. Add stripe gem `bundle add stripe`
2. Add stripe API keys to config file `EDITOR="code --wait" rails credentials:edit`

```
stripe:
  public_key: 
  secret_key: 
```

3. Create stripe initializer file `config/initializers/stripe.rb`

```ruby
Stripe.api_key = Rails.application.credentials.dig(:stripe, :secret_key)
```

1. Create stripe session in index method of listings controller

```ruby
session = Stripe::Checkout::Session.create(
    payment_method_types: ['card'],
    customer_email: current_user.email,
    line_items: [{
        name: "Donate to Musician Marketplace!",
        currency: 'aud',
        quantity: 1,
        amount: 1000
    }],
    payment_intent_data: {
        metadata: {
            user_id: current_user.id,
        }
    },
    success_url: "#{root_url}pages/donated?userId=#{current_user.id}",
    cancel_url: "#{root_url}"
)

@session_id = session.id
```

5. Create donate button in listings index view

```html
<button data-stripe="donate">
  Donate
</button>

<script src="https://js.stripe.com/v3/"></script>

<script>
  document
    .querySelector("[data-stripe='donate']")
    .addEventListener("click", () => {
      const stripe = Stripe(
        "<%= Rails.application.credentials.dig(:stripe, :public_key) %>"
      );

      stripe.redirectToCheckout({
        sessionId: "<%= @session_id %>"
      });
    });
</script>
```
6. Create donation success route / action / view

```ruby
get "/pages/donated", to: "pages#donated"

def donated
end
```

```html
Thanks for donating!
```



