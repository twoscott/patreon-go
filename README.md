# patreon-go

`patreon-go` is a Go client library for accessing the [Patreon API](https://docs.patreon.com/#api).

## How to import ##

The `patreon-go` package may be installed by running:
```
go get gopkg.in/mxpv/patreon-go.v1
```
or
```
import "gopkg.in/mxpv/patreon-go.v1"
```

## Basic example ##

```go
import "gopkg.in/mxpv/patreon-go.v1"

func main() {
	client := patreon.NewClient(nil)
  
	user, err := client.FetchUser()
	if err != nil {
		// ...
	}

	print(user.Data.Id)
}
```

## Authentication ##

The `patreon-go` library does not directly handle authentication. Instead, when creating a new client, pass an `http.Client` that can handle authentication for you, most likely you will need [oauth2](https://github.com/golang/oauth2) package.

Here is an example with static token:

```go
import (
	"gopkg.in/mxpv/patreon-go.v1"
	"golang.org/x/oauth2"
)

func NewPatreonClient(ctx context.Context, token string) *patreon.Client {
	ts := oauth2.StaticTokenSource(&oauth2.Token{AccessToken: token})
	tc := oauth2.NewClient(ctx, ts)
	
	client := patreon.NewClient(tc)
	return client
}
```

Automatically refresh token:

```go
func NewPatreonClient() (*patreon.Client, error) {
	config := oauth2.Config{
		ClientID:     "<client_id>",
		ClientSecret: "<client_secret>",
		Endpoint: oauth2.Endpoint{
			AuthURL:  AuthorizationURL,
			TokenURL: AccessTokenURL,
		},
		Scopes: []string{"users", "pledges-to-me", "my-campaign"},
	}

	token := oauth2.Token{
		AccessToken:  "<current_access_token>",
		RefreshToken: "<current_refresh_token>",
		// Must be non-nil, otherwise token will not be expired
		Expiry: time.Now().Add(-24 * time.Hour),
	}

	tc := config.Client(context.Background(), &token)

	client := NewClient(tc)
	_, err := client.FetchUser()
	if err != nil {
		panic(err)
	}

	print("OK")
}
```

## Look & Feel ##

```go
func Example_fetchPatronsAndPledges() {
	ts := oauth2.StaticTokenSource(&oauth2.Token{AccessToken: testAccessToken})
	tc := oauth2.NewClient(oauth2.NoContext, ts)

	// Create client with static access token
	client := NewClient(tc)

	// Get your campaign data
	campaignResponse, err := client.FetchCampaign()
	if err != nil {
		panic(err)
	}

	campaignId := campaignResponse.Data[0].Id

	cursor := ""
	page := 1

	for {
		pledgesResponse, err := client.FetchPledges(campaignId,
			WithPageSize(25),
			WithCursor(cursor))

		if err != nil {
			panic(err)
		}

		// Get all the users in an easy-to-lookup way
		users := make(map[string]*User)
		for _, item := range pledgesResponse.Included.Items {
			u, ok := item.(*User)
			if !ok {
				continue
			}

			users[u.Id] = u
		}

		fmt.Printf("Page %d\r\n", page)

		// Loop over the pledges to get e.g. their amount and user name
		for _, pledge := range pledgesResponse.Data {
			amount := pledge.Attributes.AmountCents
			patronId := pledge.Relationships.Patron.Data.Id
			patronFullName := users[patronId].Attributes.FullName

			fmt.Printf("%s is pledging %d cents\r\n", patronFullName, amount)
		}

		// Get the link to the next page of pledges
		nextLink := pledgesResponse.Links.Next
		if nextLink == "" {
			break
		}

		cursor = nextLink
		page++
	}

	fmt.Print("Done!")
}
```
