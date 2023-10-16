# Warn when there is a big PR
warn('Big PR') if git.lines_of_code > 500

has_app_changes = !git.modified_files.grep(/Sources/).empty?
has_test_changes = !git.modified_files.grep(/Tests/).empty?

# Add a CHANGELOG entry for app changes
if !git.modified_files.include?("CHANGELOG.md") && has_app_changes
    fail("Please include a CHANGELOG entry.")
    message "Note, we hard-wrap at 80 chars and use 2 spaces after the last line."
end

# Non-trivial amounts of app changes without tests
if git.lines_of_code > 50 && has_app_changes && !has_test_changes
    warn 'This PR may need tests.'
end