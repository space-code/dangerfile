# frozen_string_literal: true

# ==============================================================================
# General PR checks
# ==============================================================================

# Warn when there is a big PR
warn('Big PR - consider breaking it into smaller chunks') if git.lines_of_code > 500

# Encourage smaller PRs
warn('Very large PR - review may take longer') if git.lines_of_code > 1000

# Check PR has a description
# if github.pr_body.length < 10
#   fail('Please provide a meaningful PR description')
# end

# Check PR title follows conventional commits
unless github.pr_title =~ /^(feat|fix|docs|style|refactor|test|chore|perf|ci|build)(\(.+\))?:/
  warn('PR title should follow Conventional Commits format (e.g., "feat: add new feature")')
end

# ==============================================================================
# File checks
# ==============================================================================

test_files = git.modified_files + git.added_files
has_test_changes = test_files.any? { |path| path =~ %r{^Tests/.*\.swift$} }
has_app_changes = (git.modified_files + git.added_files).any? { |path| path =~ %r{^Sources/} }

has_package_changes = git.modified_files.include?('Package.swift')
has_readme_changes  = git.modified_files.include?('README.md')
has_docs_changes    = git.modified_files.grep(/\.md$/).any?

# ==============================================================================
# Tests
# ==============================================================================

# Non-trivial amounts of app changes without tests
if git.lines_of_code > 50 && has_app_changes && !has_test_changes
  warn('This PR may need tests. Consider adding test coverage.')
end

# Encourage test coverage for new files
added_swift_files = git.added_files.grep(/Sources\/.*\.swift$/)
if added_swift_files.any? && !has_test_changes
  warn("New Swift files added but no tests. Consider adding tests for: #{added_swift_files.join(', ')}")
end

# ==============================================================================
# Test file verification with project structure check
# ==============================================================================

# Helper function to check if a file exists in the project
def file_exists_in_project?(filepath)
  File.exist?(filepath)
end

# All added/modified source files
source_files = (git.modified_files + git.added_files)
                 .grep(%r{^Sources/.*\.swift$})
                 .reject { |path| File.basename(path) =~ /\+/ }

# All added/modified test files inside Tests/
test_filenames = test_files
                   .grep(%r{^Tests/.*\.swift$})
                   .map { |p| File.basename(p) }
                   .to_set

missing_tests = []
missing_test_files = []

source_files.each do |path|
  filename = File.basename(path, ".swift")
  
  next if filename.start_with?('I')
  
  expected_test_file = "#{filename}Tests.swift"

  unless test_filenames.include?(expected_test_file)
    possible_test_paths = [
      "Tests/#{expected_test_file}",
      "Tests/**/#{expected_test_file}",
      "**/Tests/**/#{expected_test_file}"
    ]
    
    test_file_exists = possible_test_paths.any? do |pattern|
      !Dir.glob(pattern).empty?
    end
    
    if test_file_exists
      missing_tests << { 
        file: filename, 
        expected: expected_test_file,
        exists: true
      }
    else
      missing_test_files << { 
        file: filename, 
        expected: expected_test_file,
        exists: false
      }
    end
  end
end

# Report missing tests (file exists but not updated)
if missing_tests.any?
  list = missing_tests
           .map { |m| "- `#{m[:file]}` → test file exists: `#{m[:expected]}` (consider updating if logic changed)" }
           .join("\n")

  message("ℹ️ Some source files have tests that weren't updated:\n#{list}")
end

# Report missing test files (file doesn't exist in project)
if missing_test_files.any?
  list = missing_test_files
           .map { |m| "- `#{m[:file]}` → expected test file: `#{m[:expected]}`" }
           .join("\n")

  warn("⚠️ Some source files are missing corresponding test files:\n#{list}")
end

# ==============================================================================
# Package.swift changes
# ==============================================================================

if has_package_changes
  message('📦 Package.swift was modified - please ensure:')
  message('- Dependencies are pinned to specific versions')
  message('- New dependencies are documented in README')
  message('- Platform requirements are correct')
  
  # Check if README was updated when adding dependencies
  package_diff = git.diff_for_file('Package.swift')
  if package_diff.patch =~ /\.package\(/ && !has_readme_changes
    warn('New dependency added but README not updated')
  end
end

# ==============================================================================
# Documentation
# ==============================================================================

# Check if public API changes include documentation updates
public_api_changes = git.diff.select { |file| 
  file.path =~ /Sources\/.*\.swift$/ && file.patch =~ /(public |open )/
}

if public_api_changes.any? && !has_docs_changes
  warn('Public API changed but no documentation updates found')
end

# ==============================================================================
# Code quality
# ==============================================================================

# Check for TODO/FIXME comments in modified files
git.modified_files.each do |file|
  next unless file.end_with?('.swift')
  
  diff = git.diff_for_file(file)
  next unless diff
  
  diff.patch.lines.each_with_index do |line, index|
    if line.start_with?('+') && (line.include?('TODO') || line.include?('FIXME'))
      warn("#{file}:#{index + 1} contains TODO/FIXME - create an issue for tracking", file: file, line: index + 1)
    end
  end
end

# Check for print statements (should use proper logging)
# Skip comments and string literals
git.modified_files.grep(/\.swift$/).each do |file|
  diff = git.diff_for_file(file)
  next unless diff
  
  diff.patch.lines.each_with_index do |line, index|
    # Skip if line is a comment (single line or doc comment)
    next if line =~ /^\+\s*(\/\/|\/\*\*)/
    
    # Check for actual print() calls (not in strings)
    if line.start_with?('+') && line =~ /\bprint\s*\(/
      warn("#{file}:#{index + 1} contains print() statement - consider using proper logging", file: file, line: index + 1)
    end
  end
end

# Check for force unwrapping in new code
git.modified_files.grep(/\.swift$/).each do |file|
  diff = git.diff_for_file(file)
  next unless diff
  
  diff.patch.lines.each_with_index do |line, index|
    if line.start_with?('+') && line =~ /!\s*(?:\)|\.|\[)/
      warn("#{file}:#{index + 1} contains force unwrapping (!) - consider using safe unwrapping", file: file, line: index + 1)
    end
  end
end

# ==============================================================================
# Generated files
# ==============================================================================

generated_files = git.modified_files.grep(/Generated/)
if generated_files.any?
  message("⚠️ Generated files were modified: #{generated_files.join(', ')}")
  message('Please verify these changes are intentional')
end

# ==============================================================================
# Success message
# ==============================================================================

if status_report[:errors].empty? && status_report[:warnings].empty?
  message('✅ Great job! All checks passed.')
end