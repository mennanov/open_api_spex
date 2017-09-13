# Open API Spex

Add Open API Specification 3 (formerly swagger) to Plug applications.

## Installation

The package can be installed by adding `open_api_spex` to your list of dependencies in `mix.exs`:

```elixir
def deps do
  [
    {:open_api_spex, github: "mbuhot/open_api_spex"}
  ]
end
```

## Usage

Start by adding an `ApiSpec` module to your application.

```elixir
defmodule MyApp.ApiSpec do
  alias OpenApiSpex.{OpenApi, Server, Info, Paths}

  def spec do
    %OpenApi{
      servers: [
        # Populate the Server info from a phoenix endpoint
        Server.from_endpoint(MyAppWeb.Endpoint, otp_app: :my_app)
      ],
      info: %Info{
        title: "My App",
        version: "1.0"
      },
      # populate the paths from a phoenix router
      paths: Paths.from_router(MyAppWeb.Router)
    }
    |> OpenApiSpex.resolve_schema_modules() # discover request/response schemas from path specs
  end
end
```

For each plug (controller) that will handle api requests, add an `open_api_operation` callback.
It will be passed the plug opts that were declared in the router, this will be the action for a phoenix controller.

```elixir
defmodule MyApp.UserController do
  alias OpenApiSpex.Operation

  @spec open_api_operation(any) :: Operation.t
  def open_api_operation(action), do: apply(__MODULE__, :"#{action}_operation", [])

  @spec show_operation() :: Operation.t
  def show_operation() do

    %Operation{
      tags: ["users"],
      summary: "Show user",
      description: "Show a user by ID",
      operationId: "UserController.show",
      parameters: [
        Operation.parameter(:id, :path, :integer, "User ID", example: 123)
      ],
      responses: %{
        200 => Operation.response("User", "application/json", Schemas.UserResponse)
      }
    }
  end
  def show(conn, %{"id" => id}) do
    {:ok, user} = MyApp.Users.find_by_id(id)
    json(conn, 200, user)
  end
end
```

Declare the JSON schemas for request/response bodies in a `Schemas` module:

```elixir
defmodule MyApp.Schemas do
  alias OpenApiSpex.Schema

  defmodule User do
    def schema do
      %Schema{
        title: "User",
        description: "A user of the app",
        type: :object,
        properties: %{
          id: %Schema{type: :integer, description: "User ID"},
          name:  %Schema{type: :string, description: "User name"},
          email: %Schema{type: :string, description: "Email address", format: :email},
          inserted_at: %Schema{type: :string, description: "Creation timestamp", format: :datetime},
          updated_at: %Schema{type: :string, description: "Update timestamp", format: :datetime}
        }
      }
    end
  end

  defmodule UserResponse do
    def schema do
      %Schema{
        title: "UserResponse",
        description: "Response schema for single user",
        type: :object,
        properties: %{
          data: User
        }
      }
    end
  end
end
```

Now you can create a mix task to write the swagger file to disk:

```elixir
defmodule Mix.Tasks.MyApp.OpenApiSpec do
  def run([output_file]) do
    json =
      MyApp.ApiSpec.spec()
      |> Poison.encode!(pretty: true)

    :ok = File.write!(output_file, json)
  end
end
```

Generate the file with: `mix myapp.openapispec spec.json`

You can also serve the swagger through a controller:

```elixir
defmodule MyApp.OpenApiSpecController do
  def show(conn, _params) do
    spec =
      MyApp.ApiSpec.spec()

    json(conn, spec)
  end
end
```

TODO: SwaggerUI 3.0

TODO: Request Validation

TODO: Validating examples in the spec

TODO: Validating responses in tests