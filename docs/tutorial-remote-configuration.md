# Remote configuration

Remote configuration is a way to customize the behavior of an app or game via in-app parameters stored on a remote server. This can be used to implement <a href="https://en.wikipedia.org/wiki/Feature_toggle" target="\_blank">feature flags</a> or adjust settings which change the appearance or behavior of the app or game.

Developers can use remote configuration to remove the hassle of a lengthy review process or modifying the game or app and then waiting for users to update. This makes it especially useful with mobile projects.

## Manage In-app parameters

The configuration settings sent to the app or game need to be stored on the server. The best way to store the information depends on how often the data will be changed.

For mostly static data it's most efficient to embed it as data structures in server-side code and for more dynamic data it's better to use a read-only [storage record](storage-collections.md).

With both of these approaches you can access remote configuration before you've done [register/login](authentication.md) or connected with a [user session](authentication.md#sessions). The in-app parameters you configure can be initialized at the earliest point of application startup.

### Static parameters

The simplest approach uses server-side code to represent the in-app parameters as a static variable. A change to the parameters after the server has started would require an update to the Lua code and a server restart.

```lua fct_label="Lua"
-- The code could be stored in a module named `"rc.lua"` and placed in the runtime path for the server.

local nk = require("nakama")

-- In-app parameters stored in a static variable.
local parameters = {
  reachable_levels = 10,
  max_player_level = 90,
  min_version = 12
}

local function remote_configuration(_context, _payload)
  return nk.json_encode({ rc = parameters })
end

nk.register_rpc(remote_configuration, "rc")
```

```go fct_label="Go"
func RemoteConfig(ctx context.Context, logger runtime.Logger, db *sql.DB, nk runtime.NakamaModule, payload string) (string, error) {
  parameters := &map[string]interface{}{
		"reachable_levels": 10,
		"max_player_level": 90,
		"min_version":      12,
	}

	if b, err := json.Marshal(parameters); err != nil {
		// Handle error
		return "", err
	} else {
		return string(b), nil
	}  
}

// Register RPC in InitModule
if err := initializer.RegisterRpc("rc", RemoteConfig); err != nil {
  logger.Error("Unable to register rc, %s", err)
  return err
}
```


### Dynamic parameters

For in-app parameters which may be changed via Analytics or with a Liveops dashboard it's more flexible to store the configuration settings in the [storage engine](storage-collections.md) as a read-only record.

```lua fct_label="Lua"
--Same as above we'll use server-side code with a module named `"rc.lua"` and placed in the runtime path for the server.

local nk = require("nakama")

local parameters = {
  reachable_levels = 10,
  max_player_level = 90,
  min_version = 12
}

local record = {
  collection = "configuration",
  record = "rc",
  user_id = nil,
  value = parameters,
  permission_read = 1,
  permission_write = 0,
  version = "*"       -- only write record if it does not already exist.
}
pcall(nk.storage_write, { record }) -- write record, ignore errors.

local function remote_configuration(_context, _payload)
  local rc = {
    collection = record.Collection,
    record = record.Record,
    user_id = nil
  }
  local records = nk.storage_read({ rc })
  return records[1].value
end

nk.register_rpc(remote_configuration, "rc")
```

```go fct_label="Go"
const COLLECTION_CONFIG = "configuration"
const KEY_CONFIG = "rc"

func SaveConfig(ctx context.Context, nk runtime.NakamaModule, logger runtime.NakamaModule) error {
	parameters := &map[string]interface{}{
		"reachable_levels": 10,
		"max_player_level": 90,
		"min_version":      12,
	}

	b, err := json.Marshal(parameters)
	if err != nil {
		// Handle error
		return err
	}

	objectIds := []*runtime.StorageWrite{
		&runtime.StorageWrite{
			Collection:      COLLECTION_CONFIG,
			Key:             KEY_CONFIG,
			Value:           string(b),
			Version:         "*", // Only write if record does not exist already
			PermissionRead:  1,
			PermissionWrite: 0,
		},
	}

	if _, err := nk.StorageWrite(ctx, objectIds); err != nil {
		// Handle error, if any
		return err
	} else {
		return nil
	}
}

func RemoteConfig(ctx context.Context, logger runtime.Logger, db *sql.DB, nk runtime.NakamaModule, payload string) (string, error) {
	objectIds := []*runtime.StorageRead{
		&runtime.StorageRead{
			Collection: COLLECTION_CONFIG,
			Key:        KEY_CONFIG,
		},
	}

	records, err := nk.StorageRead(ctx, objectIds)
	if err != nil {
		// Handle error
		return "", err
	} else {
		if len(records) == 0 {
			return "", errors.New("No config found.")
		} else {
			return records[0].Value, nil
		}
	}
}

// Register RPC in InitModule
SaveConfig(ctx, nk, logger)
if err := initializer.RegisterRpc("rc", RemoteConfig); err != nil {
  logger.Error("Unable to register rc, %s", err)
  return err
}
```

## Fetch In-app parameters

With either approach used to store in-app parameters you can fetch the configuration with a HTTP request.

!!! Tip
    Remember to change the host, port, and auth values for how you've setup your server.

```csharp fct_label="Unity"
var host = "127.0.0.1";
var port = 7350;
var path = "rc";
var auth = "defaultkey";

var format = "http://{0}:{1}/v2/rpc/{2}?http_key={3}";
var url = string.Format(format, Host, Port, Path, Auth);
var headers = new Dictionary<string, string>();
headers.Add("Content-Type", "application/json");
headers.Add("Accept", "application/json");

WWW www = new WWW(url, null, headers);
yield return www;
if (!string.IsNullOrEmpty(www.error)) {
    Debug.LogErrorFormat("Error occurred: {0}", www.error);
} else {
    var response = Encoding.UTF8.GetString(www.bytes);
    Debug.Log(response);
    // output
    // {"rc":{"max_player_level":90,"min_version":12,"reachable_levels":10}}
}
```

```shell fct_label="cURL"
curl -X POST "http://127.0.0.1:7350/v2/rpc/rc?http_key=defaultkey" \
  -H 'Content-Type: application/json' \
  -H 'Accept: application/json'
# output
# {"rc":{"max_player_level":90,"min_version":12,"reachable_levels":10}}
```
