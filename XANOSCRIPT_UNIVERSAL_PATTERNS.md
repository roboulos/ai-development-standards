# 🔧 XanoScript Universal Patterns
*Reusable patterns that work across all Xano projects*

## 🎯 Pattern Categories

1. [Authentication & Security](#authentication--security)
2. [Multi-Tenant Operations](#multi-tenant-operations)
3. [CRUD Operations](#crud-operations)
4. [Pagination & Filtering](#pagination--filtering)
5. [Error Handling](#error-handling)
6. [Activity Logging](#activity-logging)
7. [Response Formatting](#response-formatting)

---

## 🔐 Authentication & Security

### Universal Auth Check
```xanoscript
stack {
  // Always start with this
  db.get "[AUTH_TABLE]" {
    field_name = "id"
    field_value = $auth.id
  } as $current_user
  
  precondition ($current_user != null) {
    error = "Authentication required"
  }
  
  precondition ($current_user.status == "active") {
    error = "Account is not active"
  }
}
```

### Role-Based Access Pattern
```xanoscript
// Define allowed roles
var $allowed_roles = {
  value = ["admin", "manager", "owner"]
}

var $has_role = {
  value = false
}

loop ($allowed_roles.value as $role) {
  conditional {
    if ($current_user.role == $role) {
      var $has_role = {
        value = true
      }
    }
  }
}

precondition ($has_role.value == true) {
  error = "Insufficient permissions"
}
```

---

## 🏢 Multi-Tenant Operations

### Context Validation Pattern
```xanoscript
// For any resource access
db.get "[RESOURCE_TABLE]" {
  field_name = "id"
  field_value = $input.resource_id
} as $resource

precondition ($resource != null) {
  error = "Resource not found"
}

// Multi-tenant check
precondition ($resource.[TENANT_FIELD] == $current_user.[TENANT_FIELD]) {
  error = "You don't have access to this resource"
}
```

### Tenant-Scoped Queries
```xanoscript
// Build tenant filter
var $filter = {
  value = {}
}

conditional {
  if ($current_user.role != "super_admin") {
    var $filter = {
      value = {[TENANT_FIELD]: $current_user.[TENANT_FIELD]}
    }
  }
  elseif ($input.[TENANT_FIELD] != null) {
    // Super admin can specify tenant
    var $filter = {
      value = {[TENANT_FIELD]: $input.[TENANT_FIELD]}
    }
  }
}

db.query "[TABLE_NAME]" {
  filter = $filter.value
} as $results
```

---

## 📝 CRUD Operations

### Universal CREATE Pattern
```xanoscript
// Check unique constraints
db.get "[TABLE_NAME]" {
  field_name = "[UNIQUE_FIELD]"
  field_value = $input.[UNIQUE_FIELD]
} as $existing

precondition ($existing == null) {
  error = "Record with this [UNIQUE_FIELD] already exists"
}

// Create record
db.add "[TABLE_NAME]" {
  data = {
    [TENANT_FIELD]: $current_user.[TENANT_FIELD]
    // ... other fields from input
    created_by: $current_user.id
    created_at: now()
  }
} as $new_record

// Log creation
db.add "[ACTIVITY_LOG_TABLE]" {
  data = {
    [TENANT_FIELD]: $current_user.[TENANT_FIELD]
    user_id: $current_user.id
    action: "[RESOURCE]_created"
    entity_type: "[RESOURCE]"
    entity_id: $new_record.id
    created_at: now()
  }
}
```

### Universal UPDATE Pattern
```xanoscript
// Get and verify access (already done in security)
// Update only provided fields
var $update_data = {
  value = {}
}

// Build update object dynamically
conditional {
  if ($input.field1 != null) {
    var $update_data = {
      value = $update_data.value|set:"field1":$input.field1
    }
  }
  if ($input.field2 != null) {
    var $update_data = {
      value = $update_data.value|set:"field2":$input.field2
    }
  }
}

// Always add metadata
var $update_data = {
  value = $update_data.value|set:"updated_by":$current_user.id
}
var $update_data = {
  value = $update_data.value|set:"updated_at":now()
}

db.edit "[TABLE_NAME]" {
  field_name = "id"
  field_value = $input.id
  data = $update_data.value
} as $updated_record
```

### Universal SOFT DELETE Pattern
```xanoscript
// Don't actually delete, just mark
db.edit "[TABLE_NAME]" {
  field_name = "id"
  field_value = $input.id
  data = {
    status: "deleted"
    deleted_by: $current_user.id
    deleted_at: now()
  }
} as $deleted_record
```

---

## 📊 Pagination & Filtering

### Universal Pagination Pattern
```xanoscript
input {
  int page {
    description = "Page number"
    default = 1
  }
  int per_page {
    description = "Results per page"
    default = 50
    max = 100
  }
  text sort_by {
    description = "Field to sort by"
    default = "created_at"
  }
  text sort_order {
    description = "Sort direction (asc/desc)"
    default = "desc"
  }
}

// Calculate offset
var $offset = {
  value = ($input.page - 1) * $input.per_page
}

// Query with pagination
db.query "[TABLE_NAME]" {
  filter = $filter.value
  limit = $input.per_page
  offset = $offset.value
  order = $input.sort_by
  direction = $input.sort_order
} as $results

// Get total for pagination metadata
db.count "[TABLE_NAME]" {
  filter = $filter.value
} as $total_count
```

### Universal Search Pattern
```xanoscript
// Build search filter
conditional {
  if ($input.search != null) {
    // Search multiple fields
    var $search_filter = {
      value = {
        $or: [
          {name|contains: $input.search},
          {email|contains: $input.search},
          {description|contains: $input.search}
        ]
      }
    }
    
    var $filter = {
      value = $filter.value|merge:$search_filter.value
    }
  }
}
```

---

## ❌ Error Handling

### Validation Pattern
```xanoscript
// Collect all validation errors
var $errors = {
  value = []
}

conditional {
  if ($input.email != null) {
    // Email validation
    var $email_valid = {
      value = $input.email|regex:"^[^\s@]+@[^\s@]+\.[^\s@]+$"
    }
    
    if ($email_valid.value == false) {
      var $errors = {
        value = $errors.value|push:{
          field: "email"
          message: "Invalid email format"
        }
      }
    }
  }
}

conditional {
  if ($input.amount != null) {
    if ($input.amount <= 0) {
      var $errors = {
        value = $errors.value|push:{
          field: "amount"
          message: "Amount must be greater than 0"
        }
      }
    }
  }
}

// Return errors if any
conditional {
  if ($errors.value|count > 0) {
    response {
      value = {
        success: false
        errors: $errors.value
      }
    }
    stop
  }
}
```

### Not Found Pattern
```xanoscript
conditional {
  if ($resource == null) {
    response {
      value = {
        success: false
        error: {
          code: "[RESOURCE]_NOT_FOUND"
          message: "The requested [resource] does not exist"
        }
      }
    }
    stop
  }
}
```

---

## 📝 Activity Logging

### Universal Activity Log Pattern
```xanoscript
// Use after any significant action
db.add "[ACTIVITY_LOG_TABLE]" {
  data = {
    [TENANT_FIELD]: $current_user.[TENANT_FIELD]
    user_id: $current_user.id
    action: "[ACTION_NAME]"
    entity_type: "[ENTITY_TYPE]"
    entity_id: $entity.id
    details: {
      // Include relevant context
      before: $original_value
      after: $new_value
      reason: $input.reason
    }
    ip_address: $request.ip
    user_agent: $request.user_agent
    created_at: now()
  }
}
```

---

## 📤 Response Formatting

### Universal Success Response
```xanoscript
response {
  value = {
    success: true
    data: {
      [RESOURCE]: $single_item      // For single resource
      [RESOURCES]: $items_array     // For lists
    }
    meta: {
      timestamp: now()
      version: "1.0"
      [TENANT_FIELD]: $current_user.[TENANT_FIELD]
    }
  }
}
```

### Universal Paginated Response
```xanoscript
var $total_pages = {
  value = ceil($total_count / $input.per_page)
}

response {
  value = {
    success: true
    data: {
      items: $results
    }
    pagination: {
      page: $input.page
      per_page: $input.per_page
      total: $total_count
      total_pages: $total_pages.value
      has_next: $input.page < $total_pages.value
      has_prev: $input.page > 1
    }
    meta: {
      timestamp: now()
    }
  }
}
```

---

## 🔧 Utility Patterns

### UUID Generation
```xanoscript
var $new_uuid = {
  value = security.create_uuid {}
}
```

### Date Formatting
```xanoscript
var $formatted_date = {
  value = $date|date_format:"Y-m-d H:i:s"
}
```

### String Operations
```xanoscript
// Concatenation
var $full_name = {
  value = "%s %s"|sprintf:$first_name:$last_name
}

// Replace
var $processed = {
  value = $template|replace:"{{name}}":$user.name
}

// Truncate
var $summary = {
  value = $long_text|truncate:100
}
```

### Array Operations
```xanoscript
// Filter
var $active_items = {
  value = $all_items|filter:"status == 'active'"
}

// Map
var $ids = {
  value = $items|map:"id"
}

// Unique
var $unique_values = {
  value = $values|unique
}

// Count
var $total = {
  value = $items|count
}
```

---

## 📋 Implementation Checklist

When implementing any endpoint:

- [ ] Use appropriate auth pattern
- [ ] Implement multi-tenant checks
- [ ] Add input validation
- [ ] Include activity logging
- [ ] Handle errors gracefully
- [ ] Format responses consistently
- [ ] Test with different user roles
- [ ] Update documentation