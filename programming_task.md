## Original Code

```ruby
def custom_params
  {
    'Account owner' => "#{owner.name} #{owner.email}",
    'Account status' => account_details.account_status,
    'Active customers' => account_details.active_customer_count,
    'Billing system(s)' => billing_connectors.presence || ['None configured'],
    'Import status' => main_connector.try(:import_status).try(:humanize) || 'Never imported',
    'Last active' => @account.last_active_at,
    'Revenue recognition access' => @account.rev_rec_enabled?,
    'Sample Data Present?' => sample_data_present?
  }.reject { |_k, v| v.blank? }
end
```

## Issues Identified

1. **Implicit Context Dependencies**
   - Method depends on undefined context variables (`owner`, `account_details`, `@account`)
   - Makes testing and reusability difficult

2. **Null Reference Exceptions**
   - `owner.name` and `owner.email` will raise `NoMethodError` if `owner` is `nil`
   - Same issue with `account_details` methods

3. **Overly Aggressive Filtering**
   ```ruby
   .reject { |_k, v| v.blank? }
   ```
   - Removes potentially valid values: `false`, `0`, empty arrays

### 🟡 Design Issues

4. **Single Responsibility Principle Violation**
   - Mixes data collection, formatting, and filtering responsibilities

5. **Misleading Method Name**
   - `custom_params` doesn't reflect what the method actually does
   - Should indicate it's building account metadata

## Proposed Solution

```ruby
def build_account_metadata(account)
  return {} unless account
  
  metadata = {
    'Account owner' => account.owner_display_name,
    'Account status' => account.account_details&.account_status,
    'Active customers' => account.account_details&.active_customer_count,
    'Billing system(s)' => format_billing_systems(account.billing_connectors),
    'Import status' => format_import_status(account.main_connector),
    'Last active' => account.last_active_at,
    'Revenue recognition access' => account.rev_rec_enabled?,
    'Sample Data Present?' => account.sample_data_present?
  }

  yield(metadata) if block_given?
  
  # Only reject nil values, keep false/0/empty arrays
  metadata.compact
end

private

def format_billing_systems(connectors)
  connectors.presence || ['None configured']
end

def format_import_status(connector)
  connector&.import_status&.humanize || 'Never imported'
end
```

## Usage Examples

```ruby
metadata = build_account_metadata(@account)

metadata = build_account_metadata(@account) do |data|
  data['Custom field'] = 'Custom value'
  data['Account status'] = data['Account status']&.upcase
  data.delete('Last active') if hide_sensitive_data?
end
```
