---
inclusion: manual
---

# Learn Cognito with MiniStack

Amazon Cognito handles **user authentication and authorization** for your apps. It manages sign-up, sign-in, tokens, and user management so you don't have to build it yourself.

## Two Services

- **User Pools** (`cognito-idp`) — a user directory. Handles sign-up, sign-in, passwords, MFA, groups. Returns JWT tokens.
- **Identity Pools** (`cognito-identity`) — exchanges tokens for temporary AWS credentials. Lets users call AWS services directly from the browser/mobile.

Most apps only need **User Pools**. Identity Pools are for apps that need direct AWS access from the client.

## Core Concepts

- **User Pool** — your user database. One per app (or per environment).
- **App Client** — an application that can call the User Pool. Each client has its own ID and optionally a secret.
- **Auth flows** — how users authenticate. `USER_PASSWORD_AUTH` is the simplest.
- **Tokens** — Cognito returns three JWTs: `IdToken` (user info), `AccessToken` (API calls), `RefreshToken` (get new tokens).
- **Groups** — assign users to groups (e.g. `admins`, `premium`). Groups appear in the JWT.

---

## Hands-On: Create a User Pool

```python
import boto3, json

idp = boto3.client("cognito-idp",
    endpoint_url="http://localhost:4566",
    aws_access_key_id="test",
    aws_secret_access_key="test",
    region_name="us-east-1",
)

# Create a User Pool
pool = idp.create_user_pool(
    PoolName="MyAppUsers",
    Policies={
        "PasswordPolicy": {
            "MinimumLength": 8,
            "RequireUppercase": True,
            "RequireLowercase": True,
            "RequireNumbers": True,
            "RequireSymbols": False,
        }
    },
    AutoVerifiedAttributes=["email"],
    UsernameAttributes=["email"],  # users sign in with email, not username
)
pool_id = pool["UserPool"]["Id"]
print(f"Pool ID: {pool_id}")
# us-east-1_xxxxxxxxx

# Create an App Client (no secret — for browser/mobile apps)
client_resp = idp.create_user_pool_client(
    UserPoolId=pool_id,
    ClientName="web-app",
    ExplicitAuthFlows=[
        "ALLOW_USER_PASSWORD_AUTH",
        "ALLOW_REFRESH_TOKEN_AUTH",
    ],
    GenerateSecret=False,
)
client_id = client_resp["UserPoolClient"]["ClientId"]
print(f"Client ID: {client_id}")
```

---

## Hands-On: User Sign-Up and Sign-In

```python
# Self-service sign-up
idp.sign_up(
    ClientId=client_id,
    Username="alice@example.com",
    Password="Password123",
    UserAttributes=[
        {"Name": "email", "Value": "alice@example.com"},
        {"Name": "name", "Value": "Alice Smith"},
    ],
)
print("✓ User signed up")

# On real AWS, user gets a verification email.
# In MiniStack, confirm immediately:
idp.admin_confirm_sign_up(
    UserPoolId=pool_id,
    Username="alice@example.com",
)
print("✓ User confirmed")

# Sign in — get tokens
auth = idp.initiate_auth(
    ClientId=client_id,
    AuthFlow="USER_PASSWORD_AUTH",
    AuthParameters={
        "USERNAME": "alice@example.com",
        "PASSWORD": "Password123",
    },
)
tokens = auth["AuthenticationResult"]
id_token     = tokens["IdToken"]
access_token = tokens["AccessToken"]
refresh_token = tokens["RefreshToken"]
print(f"✓ Signed in — token expires in {tokens['ExpiresIn']}s")

# Decode the IdToken to see user info (it's a JWT)
import base64
payload_b64 = id_token.split(".")[1]
payload = json.loads(base64.urlsafe_b64decode(payload_b64 + "=="))
print(f"  sub: {payload['sub']}")
print(f"  email: {payload.get('email', 'N/A')}")
print(f"  cognito:username: {payload.get('cognito:username', 'N/A')}")
```

---

## Hands-On: Admin User Management

Admins can create and manage users without going through the sign-up flow.

```python
# Admin creates a user directly
idp.admin_create_user(
    UserPoolId=pool_id,
    Username="bob@example.com",
    TemporaryPassword="Temp1234!",
    UserAttributes=[
        {"Name": "email", "Value": "bob@example.com"},
        {"Name": "email_verified", "Value": "true"},
        {"Name": "name", "Value": "Bob Jones"},
    ],
    MessageAction="SUPPRESS",  # don't send welcome email
)

# Set a permanent password (skip the force-change-password flow)
idp.admin_set_user_password(
    UserPoolId=pool_id,
    Username="bob@example.com",
    Password="Permanent123",
    Permanent=True,
)
print("✓ Bob created with permanent password")

# List all users
response = idp.list_users(UserPoolId=pool_id)
for user in response["Users"]:
    email = next((a["Value"] for a in user["Attributes"] if a["Name"] == "email"), "")
    print(f"  {user['Username']} — {email} — {user['UserStatus']}")

# Get a specific user
user = idp.admin_get_user(UserPoolId=pool_id, Username="alice@example.com")
print(f"Alice status: {user['UserStatus']}")

# Update user attributes
idp.admin_update_user_attributes(
    UserPoolId=pool_id,
    Username="alice@example.com",
    UserAttributes=[
        {"Name": "custom:plan", "Value": "premium"},
    ],
)

# Disable / enable a user
idp.admin_disable_user(UserPoolId=pool_id, Username="bob@example.com")
idp.admin_enable_user(UserPoolId=pool_id, Username="bob@example.com")

# Delete a user
idp.admin_delete_user(UserPoolId=pool_id, Username="bob@example.com")
```

---

## Hands-On: Groups and Roles

Groups let you assign roles to users — great for admin/user/premium tiers.

```python
# Create groups
idp.create_group(UserPoolId=pool_id, GroupName="admins", Description="Administrators")
idp.create_group(UserPoolId=pool_id, GroupName="premium", Description="Premium users")

# Add Alice to the admins group
idp.admin_add_user_to_group(
    UserPoolId=pool_id,
    Username="alice@example.com",
    GroupName="admins",
)
idp.admin_add_user_to_group(
    UserPoolId=pool_id,
    Username="alice@example.com",
    GroupName="premium",
)

# List Alice's groups
response = idp.admin_list_groups_for_user(
    UserPoolId=pool_id,
    Username="alice@example.com",
)
for group in response["Groups"]:
    print(f"  {group['GroupName']}")
# admins
# premium

# Sign in again — groups appear in the IdToken
auth = idp.initiate_auth(
    ClientId=client_id,
    AuthFlow="USER_PASSWORD_AUTH",
    AuthParameters={
        "USERNAME": "alice@example.com",
        "PASSWORD": "Password123",
    },
)
id_token = auth["AuthenticationResult"]["IdToken"]
payload = json.loads(base64.urlsafe_b64decode(id_token.split(".")[1] + "=="))
print(f"Groups in token: {payload.get('cognito:groups', [])}")
# ['admins', 'premium']
```

---

## Hands-On: Token Refresh

Access tokens expire (default 1 hour). Use the refresh token to get new ones.

```python
# Refresh tokens without re-entering password
refresh = idp.initiate_auth(
    ClientId=client_id,
    AuthFlow="REFRESH_TOKEN_AUTH",
    AuthParameters={"REFRESH_TOKEN": refresh_token},
)
new_access_token = refresh["AuthenticationResult"]["AccessToken"]
print("✓ Tokens refreshed")
```

---

## Hands-On: Password Reset Flow

```python
# User forgot their password — triggers email with code
# (MiniStack accepts any code)
idp.forgot_password(
    ClientId=client_id,
    Username="alice@example.com",
)

# User enters the code from their email + new password
idp.confirm_forgot_password(
    ClientId=client_id,
    Username="alice@example.com",
    ConfirmationCode="123456",  # any code works in MiniStack
    Password="NewPassword456",
)
print("✓ Password reset")

# Sign in with new password
auth = idp.initiate_auth(
    ClientId=client_id,
    AuthFlow="USER_PASSWORD_AUTH",
    AuthParameters={
        "USERNAME": "alice@example.com",
        "PASSWORD": "NewPassword456",
    },
)
print(f"✓ Signed in with new password")
```

---

## Hands-On: Get User Info (Self-Service)

```python
# Get the current user's info using their access token
user_info = idp.get_user(AccessToken=access_token)
print(f"Username: {user_info['Username']}")
for attr in user_info["UserAttributes"]:
    print(f"  {attr['Name']}: {attr['Value']}")

# Update own attributes
idp.update_user_attributes(
    AccessToken=access_token,
    UserAttributes=[
        {"Name": "name", "Value": "Alice Johnson"},
    ],
)

# Change own password
idp.change_password(
    AccessToken=access_token,
    PreviousPassword="NewPassword456",
    ProposedPassword="AnotherPassword789",
)

# Sign out everywhere
idp.global_sign_out(AccessToken=access_token)
```

---

## Hands-On: Protect an API with Cognito

The full pattern — Cognito + API Gateway + Lambda.

```python
import zipfile, io

lam = boto3.client("lambda",
    endpoint_url="http://localhost:4566",
    aws_access_key_id="test",
    aws_secret_access_key="test",
    region_name="us-east-1",
)
apigw = boto3.client("apigatewayv2",
    endpoint_url="http://localhost:4566",
    aws_access_key_id="test",
    aws_secret_access_key="test",
    region_name="us-east-1",
)

# Lambda that reads user info from the JWT claims
def make_zip(code):
    buf = io.BytesIO()
    with zipfile.ZipFile(buf, "w") as zf:
        zf.writestr("index.py", code)
    return buf.getvalue()

lam.create_function(
    FunctionName="protected-api",
    Runtime="python3.12",
    Role="arn:aws:iam::000000000000:role/lambda-role",
    Handler="index.handler",
    Code={"ZipFile": make_zip("""
import json, base64

def handler(event, context):
    # On real AWS with a JWT authorizer, claims come from:
    # event["requestContext"]["authorizer"]["jwt"]["claims"]
    # In MiniStack, decode the token manually from the Authorization header
    auth_header = (event.get("headers") or {}).get("authorization", "")
    token = auth_header.replace("Bearer ", "")
    
    claims = {}
    if token:
        try:
            payload_b64 = token.split(".")[1]
            claims = json.loads(base64.urlsafe_b64decode(payload_b64 + "=="))
        except Exception:
            pass
    
    user_id = claims.get("sub", "anonymous")
    groups = claims.get("cognito:groups", [])
    
    return {
        "statusCode": 200,
        "body": json.dumps({
            "message": f"Hello, {user_id}!",
            "groups": groups,
            "isAdmin": "admins" in groups,
        }),
    }
""")},
)

fn_arn = "arn:aws:lambda:us-east-1:000000000000:function:protected-api"

# Create API with JWT authorizer
api = apigw.create_api(Name="protected-api", ProtocolType="HTTP")
api_id = api["ApiId"]

# JWT authorizer — validates Cognito tokens
issuer = f"https://cognito-idp.us-east-1.amazonaws.com/{pool_id}"
authorizer = apigw.create_authorizer(
    ApiId=api_id,
    AuthorizerType="JWT",
    Name="cognito-auth",
    IdentitySource=["$request.header.Authorization"],
    JwtConfiguration={
        "Issuer": issuer,
        "Audience": [client_id],
    },
)
auth_id = authorizer["AuthorizerId"]

integration = apigw.create_integration(
    ApiId=api_id,
    IntegrationType="AWS_PROXY",
    IntegrationUri=fn_arn,
    PayloadFormatVersion="2.0",
)
int_id = integration["IntegrationId"]

# Protected route — requires valid JWT
apigw.create_route(
    ApiId=api_id,
    RouteKey="GET /me",
    Target=f"integrations/{int_id}",
    AuthorizationType="JWT",
    AuthorizerId=auth_id,
)

apigw.create_stage(ApiId=api_id, StageName="$default")

# Sign in and call the protected API
auth = idp.initiate_auth(
    ClientId=client_id,
    AuthFlow="USER_PASSWORD_AUTH",
    AuthParameters={
        "USERNAME": "alice@example.com",
        "PASSWORD": "AnotherPassword789",
    },
)
access_token = auth["AuthenticationResult"]["AccessToken"]

# Call the API with the token
import urllib.request
req = urllib.request.Request(
    f"http://{api_id}.execute-api.localhost:4566/me",
    headers={"Authorization": f"Bearer {access_token}"},
)
with urllib.request.urlopen(req) as resp:
    data = json.loads(resp.read())
    print(f"User: {data['message']}")
    print(f"Groups: {data['groups']}")
    print(f"Is admin: {data['isAdmin']}")
```

---

## Hands-On: Identity Pools (AWS Credentials from Token)

Identity Pools let authenticated users get temporary AWS credentials to call services directly.

```python
identity = boto3.client("cognito-identity",
    endpoint_url="http://localhost:4566",
    aws_access_key_id="test",
    aws_secret_access_key="test",
    region_name="us-east-1",
)

# Create an Identity Pool
id_pool = identity.create_identity_pool(
    IdentityPoolName="MyAppIdentityPool",
    AllowUnauthenticatedIdentities=False,
    CognitoIdentityProviders=[{
        "ProviderName": f"cognito-idp.us-east-1.amazonaws.com/{pool_id}",
        "ClientId": client_id,
    }],
)
identity_pool_id = id_pool["IdentityPoolId"]

# Exchange Cognito token for an identity ID
id_resp = identity.get_id(
    IdentityPoolId=identity_pool_id,
    Logins={
        f"cognito-idp.us-east-1.amazonaws.com/{pool_id}": id_token,
    },
)
identity_id = id_resp["IdentityId"]

# Get temporary AWS credentials
creds_resp = identity.get_credentials_for_identity(
    IdentityId=identity_id,
    Logins={
        f"cognito-idp.us-east-1.amazonaws.com/{pool_id}": id_token,
    },
)
creds = creds_resp["Credentials"]
print(f"AccessKeyId: {creds['AccessKeyId']}")
print(f"Expiration: {creds['Expiration']}")

# Use these credentials to call AWS services directly
s3 = boto3.client("s3",
    endpoint_url="http://localhost:4566",
    aws_access_key_id=creds["AccessKeyId"],
    aws_secret_access_key=creds["SecretKey"],
    aws_session_token=creds["SessionToken"],
)
```

---

## What's Different on Real AWS

- **Email verification** — `SignUp` sends a real verification email. Users must click the link before they can sign in. MiniStack skips this — use `admin_confirm_sign_up` in tests.
- **Password reset codes** — `ForgotPassword` sends a real email with a 6-digit code. MiniStack accepts any code.
- **JWT validation** — on real AWS, API Gateway validates the JWT signature against Cognito's JWKS endpoint. MiniStack stores the authorizer config but doesn't enforce it.
- **MFA** — Cognito supports SMS and TOTP MFA. MiniStack stores MFA config but doesn't enforce it.
- **Hosted UI** — Cognito provides a pre-built login page at `https://{domain}.auth.{region}.amazoncognito.com`. MiniStack has a basic OAuth2 endpoint at `/oauth2/token`.
- **Social login** — Cognito supports Google, Facebook, Apple, SAML. MiniStack has SAML stub support.
- **Pricing** — first 50,000 MAU free, then $0.0055/MAU. Essentially free for most apps.

## Next Steps

- Add Cognito auth to the **Serverless REST API** pattern — replace the `X-User-Id` header with a real JWT authorizer.
- Use **groups** to implement role-based access control (RBAC) in your Lambda.
- Try **custom attributes** (`custom:plan`, `custom:org_id`) to store app-specific user data.
