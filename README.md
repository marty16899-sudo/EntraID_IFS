# Access Package API Backend

Backend API for Microsoft Teams app to manage Entra ID Access Package requests.

## Features

- List available access packages
- Search for access packages
- Submit access package requests
- Track request status
- View current assignments
- On-Behalf-Of (OBO) authentication flow
- Secure token validation

## Prerequisites

- Node.js 18+ 
- Azure AD App Registration with the following permissions:
  - `EntitlementManagement.Read.All`
  - `EntitlementManagement.ReadWrite.All`
  - `User.Read`
  - `User.Read.All`

## Setup

### 1. Install Dependencies

```bash
npm install
```

### 2. Configure Environment Variables

Copy `.env.example` to `.env` and fill in your values:

```bash
cp .env.example .env
```

Required variables:
- `TENANT_ID`: Your Azure AD tenant ID
- `CLIENT_ID`: Your app registration client ID
- `CLIENT_SECRET`: Your app registration client secret
- `PORT`: Server port (default: 3001)

### 3. Azure App Registration Setup

1. Go to Azure Portal → Azure Active Directory → App registrations
2. Create a new registration or use existing
3. Under "Certificates & secrets", create a new client secret
4. Under "API permissions", add:
   - Microsoft Graph → Delegated → EntitlementManagement.Read.All
   - Microsoft Graph → Delegated → EntitlementManagement.ReadWrite.All
   - Microsoft Graph → Delegated → User.Read
   - Microsoft Graph → Delegated → User.Read.All
5. Grant admin consent for these permissions
6. Under "Authentication", add redirect URIs for your Teams app
7. Under "Expose an API", add a scope for your Teams app

### 4. Run the Server

Development mode (with auto-reload):
```bash
npm run dev
```

Production mode:
```bash
npm start
```

## API Endpoints

### Authentication
- `POST /api/auth/validate` - Validate token and get user info
- `GET /api/auth/health` - Check auth service configuration

### Access Packages
- `GET /api/access-packages` - List all available access packages
- `GET /api/access-packages/search?q=term` - Search access packages
- `GET /api/access-packages/:id` - Get access package details

### Access Requests
- `POST /api/access-requests` - Submit new access request
- `GET /api/access-requests` - Get user's requests
- `GET /api/access-requests/:id` - Get request details
- `DELETE /api/access-requests/:id` - Cancel a request

### User Access
- `GET /api/user-access/assignments` - Get user's current assignments
- `GET /api/user-access/profile` - Get user profile

## Request Examples

### List Access Packages
```bash
curl -X GET http://localhost:3001/api/access-packages \
  -H "Authorization: Bearer YOUR_TOKEN"
```

### Submit Access Request
```bash
curl -X POST http://localhost:3001/api/access-requests \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "packageId": "package-guid",
    "policyId": "policy-guid",
    "justification": "I need access for project work"
  }'
```

### Get User's Requests
```bash
curl -X GET http://localhost:3001/api/access-requests \
  -H "Authorization: Bearer YOUR_TOKEN"
```

## Architecture

```
access-package-api/
├── server.js                 # Main Express server
├── package.json             # Dependencies
├── .env                     # Environment variables (not in git)
├── .env.example            # Environment template
├── services/
│   └── graphService.js     # Microsoft Graph API client
├── middleware/
│   └── auth.js             # Authentication middleware
└── routes/
    ├── auth.js             # Auth routes
    ├── accessPackages.js   # Access package routes
    ├── accessRequests.js   # Request management routes
    └── userAccess.js       # User access routes
```

## Security

- All routes require valid JWT token from Azure AD
- On-Behalf-Of (OBO) flow ensures user context is maintained
- Token validation using JWKS
- CORS configuration to allow only specific origins
- Helmet.js for security headers

## Error Handling

All errors return JSON in this format:
```json
{
  "error": {
    "message": "Error description",
    "code": "ERROR_CODE"
  }
}
```

Common error codes:
- `UNAUTHORIZED` - Missing or invalid token
- `FORBIDDEN` - Insufficient permissions
- `MISSING_PARAMETERS` - Required fields not provided
- `GRAPH_API_ERROR` - Microsoft Graph API error
- `INTERNAL_ERROR` - Server error

## Troubleshooting

### Token validation fails
- Ensure your `TENANT_ID` and `CLIENT_ID` are correct
- Check that the token audience matches your `CLIENT_ID`

### Graph API permission errors
- Verify permissions are granted in Azure Portal
- Ensure admin consent is granted
- Check that you're using the beta endpoint for entitlement management

### OBO flow fails
- Verify `CLIENT_SECRET` is correct and not expired
- Ensure the app has permission to use OBO flow
- Check that the incoming token has the correct scopes

## Development Tips

1. Use `npm run dev` for hot reloading during development
2. Check `/api/auth/health` to verify configuration
3. Use the `/api/auth/validate` endpoint to test token validation
4. Enable detailed logging by setting `NODE_ENV=development`

## Production Deployment

For production:
1. Use environment variables (don't commit `.env`)
2. Enable HTTPS
3. Configure CORS with specific origins
4. Set up monitoring and logging
5. Use a process manager like PM2
6. Consider deploying to Azure App Service or Azure Functions

## License

MIT
