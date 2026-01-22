# Metadata Schema Proposal

## Overview
Define a schema for metadata fields that Stripe entities (Customer, Subscription, etc.) should have. This schema will be stored in the tenant's `config` JSONB column and used to dynamically generate form fields.

## Schema Structure

### Proposed JSON Structure
```json
{
  "metadata_schema": {
    "customer": [
      {
        "key": "first_name",
        "label": "First Name",
        "type": "string",
        "required": true,
        "maxLength": 50,
        "order": 1,
        "group": "personal_info"
      },
      {
        "key": "last_name",
        "label": "Last Name",
        "type": "string",
        "required": true,
        "maxLength": 50,
        "order": 2,
        "group": "personal_info"
      },
      {
        "key": "member_id",
        "label": "Member ID",
        "type": "string",
        "required": false,
        "pattern": "^[A-Z0-9]{6,10}$",
        "order": 3,
        "group": "membership"
      },
      {
        "key": "is_committee",
        "label": "Committee Member",
        "type": "boolean",
        "required": false,
        "default": false,
        "order": 4,
        "group": "membership"
      },
      {
        "key": "join_date",
        "label": "Join Date",
        "type": "date",
        "required": false,
        "order": 5,
        "group": "membership"
      }
    ],
    "subscription": [
      {
        "key": "FY",
        "label": "Financial Year",
        "type": "string",
        "required": false,
        "pattern": "^FY\\d{4}$",
        "order": 1,
        "group": "billing"
      },
      {
        "key": "description",
        "label": "Description",
        "type": "text",
        "required": false,
        "maxLength": 500,
        "order": 2,
        "group": "billing"
      },
      {
        "key": "auto_renew",
        "label": "Auto Renew",
        "type": "boolean",
        "required": false,
        "default": true,
        "order": 3,
        "group": "billing"
      }
    ],
    "payment_method": [
      {
        "key": "preferred",
        "label": "Preferred Payment Method",
        "type": "boolean",
        "required": false,
        "default": false,
        "order": 1
      }
    ]
  }
}
```

## Field Properties

### Required Properties
- `key`: The metadata key name (e.g., "first_name")
- `label`: Display label for the field
- `type`: Field type (see types below)
- `order`: Display order (lower numbers first)

### Optional Properties
- `required`: Boolean, whether field is required (default: false)
- `group`: String, group name for organizing fields
- `default`: Default value for the field
- `maxLength`: Maximum length for string/text fields
- `minLength`: Minimum length for string/text fields
- `pattern`: Regex pattern for validation
- `options`: Array of options for select/dropdown fields
- `placeholder`: Placeholder text
- `helpText`: Help text to display below field
- `validation`: Custom validation function name or rules

## Field Types

### Supported Types
- `string`: Single-line text input
- `text`: Multi-line textarea
- `number`: Numeric input
- `boolean`: Checkbox or toggle
- `date`: Date picker
- `email`: Email input with validation
- `url`: URL input with validation
- `select`: Dropdown select
- `multiselect`: Multi-select dropdown

## Alternative Simpler Structure

If you prefer a simpler structure:

```json
{
  "metadata_schema": {
    "customer": [
      {
        "name": "first_name",
        "label": "First Name",
        "type": "string",
        "required": true
      },
      {
        "name": "last_name",
        "label": "Last Name",
        "type": "string",
        "required": true
      }
    ],
    "subscription": [
      {
        "name": "FY",
        "label": "Financial Year",
        "type": "string"
      },
      {
        "name": "description",
        "label": "Description",
        "type": "text"
      }
    ]
  }
}
```

## Implementation Plan

1. **Database**: Store in `tenants.config` JSONB column
2. **API Endpoints**:
   - `GET /api/{tenant_id}/metadata-schema` - Get schema
   - `PUT /api/{tenant_id}/metadata-schema` - Update schema
   - `GET /api/{tenant_id}/metadata-schema/{entity_type}` - Get schema for specific entity
3. **Frontend Components**:
   - `MetadataForm` component that generates form fields from schema
   - `MetadataSchemaEditor` component for admins to edit schema
4. **Usage**: Use in CustomerStripeCustomerPage, Subscription pages, etc.

## Benefits

1. **Flexible**: Each tenant can define their own metadata structure
2. **Type-safe**: Field types ensure correct data format
3. **Validated**: Built-in validation rules
4. **User-friendly**: Labels and help text improve UX
5. **Maintainable**: Centralized schema definition
6. **Extensible**: Easy to add new field types or properties
