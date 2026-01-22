# UI Standards and Templates

## Table of Contents

1. [Design System](#design-system)
2. [Color Palette](#color-palette)
3. [Typography](#typography)
4. [Component Templates](#component-templates)
   - [Header](#header)
   - [Footer](#footer)
   - [Landing Page](#landing-page)
   - [Paginated Table](#paginated-table)
   - [Delete Record](#delete-record)
   - [Update Record](#update-record)
   - [Insert Record](#insert-record)
   - [Static Content Page](#static-content-page)
5. [Layout Patterns](#layout-patterns)
6. [Responsive Design](#responsive-design)

---

## Design System

### Overview

Cogento uses the **Edgible look and feel** with Material Design components and Tailwind CSS for styling.

**Key Design Principles:**
- Clean, modern interface
- Consistent color scheme
- Accessible design
- Responsive layout
- Material Design components for consistency

---

## Color Palette

### Primary Colors

```css
/* Edgible Brand Colors */
--edgible-dark-blue: #000033;  /* Headers, footers, primary actions */
--edgible-white: #FFFFFF;      /* Page backgrounds */
--edgible-text-dark: #212121; /* Primary text */
--edgible-text-light: #757575; /* Secondary text */
```

**Usage:**
- **Headers and Footers:** Background color `#000033`
- **Page Background:** White (`#FFFFFF`)
- **Text:** Dark gray (`#212121`) for primary text, light gray (`#757575`) for secondary

### Material Design Colors

```css
/* Material Design Primary */
--md-primary: #1976d2;
--md-primary-dark: #1565c0;
--md-primary-light: #42a5f5;

/* Material Design Accent */
--md-accent: #ff4081;

/* Material Design Status */
--md-success: #4caf50;
--md-warning: #ff9800;
--md-error: #f44336;
--md-info: #2196f3;
```

---

## Typography

### Font Stack

```css
font-family: 'Roboto', 'Helvetica Neue', Arial, sans-serif;
```

### Type Scale

```css
/* Headings */
h1 { font-size: 2.5rem; font-weight: 500; }
h2 { font-size: 2rem; font-weight: 500; }
h3 { font-size: 1.75rem; font-weight: 500; }
h4 { font-size: 1.5rem; font-weight: 500; }
h5 { font-size: 1.25rem; font-weight: 500; }
h6 { font-size: 1rem; font-weight: 500; }

/* Body */
body { font-size: 1rem; font-weight: 400; line-height: 1.5; }
small { font-size: 0.875rem; }
```

---

## Component Templates

### Header

**Purpose:** Site-wide navigation header with Edgible branding.

**Design:**
- Background: `#000033` (Edgible dark blue)
- Text: White
- Fixed position at top
- Responsive navigation menu

**Template:**

```typescript
// components/Header.tsx
import React, { useState, useRef, useEffect } from 'react';
import { Link, useNavigate } from 'react-router-dom';
import { useAuth } from '../contexts/AuthContext';
import { useClub } from '../contexts/ClubContext';
import { TenantSwitcher } from './TenantSwitcher';

export function Header() {
  const { user, signOut } = useAuth();
  const { clubId, clubName } = useClub();
  const navigate = useNavigate();
  const [menuOpen, setMenuOpen] = useState(false);
  const menuRef = useRef<HTMLDivElement>(null);

  // Close menu when clicking outside
  useEffect(() => {
    const handleClickOutside = (event: MouseEvent) => {
      if (menuRef.current && !menuRef.current.contains(event.target as Node)) {
        setMenuOpen(false);
      }
    };

    if (menuOpen) {
      document.addEventListener('mousedown', handleClickOutside);
    }

    return () => {
      document.removeEventListener('mousedown', handleClickOutside);
    };
  }, [menuOpen]);

  const handleSignOut = () => {
    signOut();
    navigate('/');
    setMenuOpen(false);
  };

  return (
    <header className="bg-[#000033] text-white shadow-md relative">
      <div className="container mx-auto px-4 py-4">
        <div className="flex items-center justify-between">
          {/* Logo/Brand */}
          <div className="flex items-center space-x-4">
            <Link to={`/${clubId}/dashboard`} className="text-xl font-bold">
              Cogento
            </Link>
            {clubName && (
              <span className="text-sm opacity-90 hidden md:inline">
                {clubName}
              </span>
            )}
          </div>

          {/* Desktop Navigation */}
          <nav className="hidden md:flex items-center space-x-6">
            {user && (
              <>
                <Link 
                  to={`/${clubId}/dashboard`}
                  className="hover:opacity-80 transition-opacity"
                >
                  Dashboard
                </Link>
                <Link 
                  to={`/${clubId}/members`}
                  className="hover:opacity-80 transition-opacity"
                >
                  Members
                </Link>
                <Link 
                  to={`/${clubId}/subscriptions`}
                  className="hover:opacity-80 transition-opacity"
                >
                  Subscriptions
                </Link>
                {user.isSuperuser && <TenantSwitcher />}
                <button
                  onClick={handleSignOut}
                  className="hover:opacity-80 transition-opacity"
                >
                  Sign Out
                </button>
              </>
            )}
          </nav>

          {/* Mobile Menu Button */}
          {user && (
            <div className="md:hidden relative" ref={menuRef}>
              <button
                onClick={() => setMenuOpen(!menuOpen)}
                className="p-2 rounded-lg hover:bg-white/10 transition-colors focus:outline-none focus:ring-2 focus:ring-white/50"
                aria-label="Menu"
                aria-expanded={menuOpen}
              >
                {/* Material Design 3-bar hamburger icon */}
                <svg
                  className="w-6 h-6"
                  fill="none"
                  stroke="currentColor"
                  viewBox="0 0 24 24"
                  xmlns="http://www.w3.org/2000/svg"
                >
                  <path
                    strokeLinecap="round"
                    strokeLinejoin="round"
                    strokeWidth={2}
                    d="M4 6h16M4 12h16M4 18h16"
                  />
                </svg>
              </button>

              {/* Dropdown Menu */}
              {menuOpen && (
                <div className="absolute right-0 mt-2 w-48 bg-white rounded-lg shadow-xl z-50 border border-gray-200">
                  <div className="py-2">
                    <Link
                      to={`/${clubId}/dashboard`}
                      onClick={() => setMenuOpen(false)}
                      className="block px-4 py-2 text-gray-900 hover:bg-gray-100 transition-colors"
                    >
                      Dashboard
                    </Link>
                    <Link
                      to={`/${clubId}/members`}
                      onClick={() => setMenuOpen(false)}
                      className="block px-4 py-2 text-gray-900 hover:bg-gray-100 transition-colors"
                    >
                      Members
                    </Link>
                    <Link
                      to={`/${clubId}/subscriptions`}
                      onClick={() => setMenuOpen(false)}
                      className="block px-4 py-2 text-gray-900 hover:bg-gray-100 transition-colors"
                    >
                      Subscriptions
                    </Link>
                    {user.isSuperuser && (
                      <div className="px-4 py-2 border-t border-gray-200">
                        <TenantSwitcher />
                      </div>
                    )}
                    <div className="border-t border-gray-200 mt-2 pt-2">
                      <button
                        onClick={handleSignOut}
                        className="block w-full text-left px-4 py-2 text-gray-900 hover:bg-gray-100 transition-colors"
                      >
                        Sign Out
                      </button>
                    </div>
                  </div>
                </div>
              )}
            </div>
          )}
        </div>
      </div>
    </header>
  );
}
```

**Features:**
- Hamburger menu button (3-bar Material Design icon) on the right side
- Dropdown menu appears on click
- Closes when clicking outside
- Responsive: full navigation on desktop, hamburger menu on mobile
- Material Design icon styling

**Styling (Tailwind):**
```css
/* Header specific styles */
header {
  background-color: #000033;
  color: white;
  box-shadow: 0 2px 4px rgba(0,0,0,0.1);
}

/* Hamburger menu button */
.menu-button {
  padding: 0.5rem;
  border-radius: 0.5rem;
  transition: background-color 0.2s;
}

.menu-button:hover {
  background-color: rgba(255, 255, 255, 0.1);
}

/* Dropdown menu */
.dropdown-menu {
  background: white;
  border-radius: 0.5rem;
  box-shadow: 0 10px 15px -3px rgba(0, 0, 0, 0.1);
  z-index: 50;
}
```

---

### Footer

**Purpose:** Site-wide footer with links and copyright.

**Design:**
- Background: `#000033` (Edgible dark blue)
- Text: White
- Fixed position at bottom (or relative)

**Template:**

```typescript
// components/Footer.tsx
import React from 'react';
import { Link } from 'react-router-dom';
import { useClub } from '../contexts/ClubContext';

export function Footer() {
  const { clubId } = useClub();

  return (
    <footer className="bg-[#000033] text-white mt-auto">
      <div className="container mx-auto px-4 py-8">
        <div className="grid grid-cols-1 md:grid-cols-3 gap-8">
          {/* About */}
          <div>
            <h3 className="font-semibold mb-4">About</h3>
            <ul className="space-y-2 text-sm">
              <li>
                <Link 
                  to={`/${clubId}/about`}
                  className="hover:opacity-80 transition-opacity"
                >
                  About Us
                </Link>
              </li>
              <li>
                <Link 
                  to={`/${clubId}/help`}
                  className="hover:opacity-80 transition-opacity"
                >
                  Help
                </Link>
              </li>
            </ul>
          </div>

          {/* Legal */}
          <div>
            <h3 className="font-semibold mb-4">Legal</h3>
            <ul className="space-y-2 text-sm">
              <li>
                <Link 
                  to={`/${clubId}/privacy`}
                  className="hover:opacity-80 transition-opacity"
                >
                  Privacy Policy
                </Link>
              </li>
              <li>
                <Link 
                  to={`/${clubId}/terms`}
                  className="hover:opacity-80 transition-opacity"
                >
                  Terms of Service
                </Link>
              </li>
            </ul>
          </div>

          {/* Contact */}
          <div>
            <h3 className="font-semibold mb-4">Contact</h3>
            <p className="text-sm">
              Email: support@cogento.com
            </p>
          </div>
        </div>

        {/* Copyright */}
        <div className="mt-8 pt-8 border-t border-white/20 text-center text-sm">
          <p>&copy; {new Date().getFullYear()} Cogento. All rights reserved.</p>
        </div>
      </div>
    </footer>
  );
}
```

---

### Landing Page

**Purpose:** Initial page users see (before authentication or tenant selection).

**Design:**
- White background
- Centered content
- Call-to-action buttons
- Edgible branding

**Template:**

```typescript
// pages/LandingPage.tsx
import React from 'react';
import { Link } from 'react-router-dom';
import { Header } from '../components/Header';
import { Footer } from '../components/Footer';

export function LandingPage() {
  return (
    <div className="min-h-screen flex flex-col bg-white">
      <Header />
      
      <main className="flex-grow">
        {/* Hero Section */}
        <section className="py-20 px-4">
          <div className="container mx-auto text-center">
            <h1 className="text-4xl md:text-5xl font-bold text-gray-900 mb-6">
              Self-hosted Membership Management
            </h1>
            <p className="text-xl text-gray-600 mb-8 max-w-2xl mx-auto">
              Manage your club members, subscriptions, and payments with Cogento.
              Built on Stripe, powered by you.
            </p>
            <div className="flex justify-center space-x-4">
              <Link
                to="/signin"
                className="bg-[#000033] text-white px-8 py-3 rounded-lg font-semibold hover:opacity-90 transition-opacity"
              >
                Sign In
              </Link>
              <Link
                to="/signup"
                className="bg-gray-200 text-gray-900 px-8 py-3 rounded-lg font-semibold hover:bg-gray-300 transition-colors"
              >
                Get Started
              </Link>
            </div>
          </div>
        </section>

        {/* Features Section */}
        <section className="py-16 px-4 bg-gray-50">
          <div className="container mx-auto">
            <h2 className="text-3xl font-bold text-center mb-12">
              Features
            </h2>
            <div className="grid grid-cols-1 md:grid-cols-3 gap-8">
              <div className="text-center">
                <h3 className="text-xl font-semibold mb-4">Stripe Integration</h3>
                <p className="text-gray-600">
                  Stripe as your single source of truth for all membership data.
                </p>
              </div>
              <div className="text-center">
                <h3 className="text-xl font-semibold mb-4">Self-Hosted</h3>
                <p className="text-gray-600">
                  Complete control over your data and infrastructure.
                </p>
              </div>
              <div className="text-center">
                <h3 className="text-xl font-semibold mb-4">Multi-Tenant</h3>
                <p className="text-gray-600">
                  Support multiple clubs in a single deployment.
                </p>
              </div>
            </div>
          </div>
        </section>
      </main>

      <Footer />
    </div>
  );
}
```

---

### Paginated Table

**Purpose:** Display tabular data with pagination controls.

**Design:**
- Clean table layout
- Pagination controls at bottom
- Search/filter capabilities
- Responsive design

**Template:**

```typescript
// components/PaginatedTable.tsx
import React, { useState, useEffect } from 'react';

interface PaginatedTableProps<T> {
  data: T[];
  columns: {
    key: keyof T;
    label: string;
    render?: (value: any, row: T) => React.ReactNode;
  }[];
  itemsPerPage?: number;
  onRowClick?: (row: T) => void;
  searchable?: boolean;
  searchPlaceholder?: string;
}

export function PaginatedTable<T extends Record<string, any>>({
  data,
  columns,
  itemsPerPage = 10,
  onRowClick,
  searchable = false,
  searchPlaceholder = "Search...",
}: PaginatedTableProps<T>) {
  const [currentPage, setCurrentPage] = useState(1);
  const [searchTerm, setSearchTerm] = useState('');

  // Filter data based on search
  const filteredData = searchable && searchTerm
    ? data.filter(row =>
        columns.some(col => {
          const value = row[col.key];
          return value?.toString().toLowerCase().includes(searchTerm.toLowerCase());
        })
      )
    : data;

  // Calculate pagination
  const totalPages = Math.ceil(filteredData.length / itemsPerPage);
  const startIndex = (currentPage - 1) * itemsPerPage;
  const endIndex = startIndex + itemsPerPage;
  const paginatedData = filteredData.slice(startIndex, endIndex);

  // Reset to page 1 when search changes
  useEffect(() => {
    setCurrentPage(1);
  }, [searchTerm]);

  return (
    <div className="bg-white rounded-lg shadow-md overflow-hidden">
      {/* Search Bar */}
      {searchable && (
        <div className="p-4 border-b">
          <input
            type="text"
            placeholder={searchPlaceholder}
            value={searchTerm}
            onChange={(e) => setSearchTerm(e.target.value)}
            className="w-full px-4 py-2 border border-gray-300 rounded-lg focus:outline-none focus:ring-2 focus:ring-[#000033]"
          />
        </div>
      )}

      {/* Table */}
      <div className="overflow-x-auto">
        <table className="w-full">
          <thead className="bg-gray-50">
            <tr>
              {columns.map((column) => (
                <th
                  key={String(column.key)}
                  className="px-6 py-3 text-left text-xs font-medium text-gray-500 uppercase tracking-wider"
                >
                  {column.label}
                </th>
              ))}
            </tr>
          </thead>
          <tbody className="bg-white divide-y divide-gray-200">
            {paginatedData.length === 0 ? (
              <tr>
                <td
                  colSpan={columns.length}
                  className="px-6 py-8 text-center text-gray-500"
                >
                  No data available
                </td>
              </tr>
            ) : (
              paginatedData.map((row, index) => (
                <tr
                  key={index}
                  onClick={() => onRowClick?.(row)}
                  className={onRowClick ? 'cursor-pointer hover:bg-gray-50 transition-colors' : ''}
                >
                  {columns.map((column) => (
                    <td
                      key={String(column.key)}
                      className="px-6 py-4 whitespace-nowrap text-sm text-gray-900"
                    >
                      {column.render
                        ? column.render(row[column.key], row)
                        : row[column.key]}
                    </td>
                  ))}
                </tr>
              ))
            )}
          </tbody>
        </table>
      </div>

      {/* Pagination Controls */}
      <div className="px-6 py-4 border-t bg-gray-50 flex items-center justify-between">
        <div className="text-sm text-gray-700">
          Showing {startIndex + 1} to {Math.min(endIndex, filteredData.length)} of{' '}
          {filteredData.length} results
        </div>
        <div className="flex space-x-2">
          <button
            onClick={() => setCurrentPage(prev => Math.max(1, prev - 1))}
            disabled={currentPage === 1}
            className="px-4 py-2 border border-gray-300 rounded-lg disabled:opacity-50 disabled:cursor-not-allowed hover:bg-gray-100"
          >
            Previous
          </button>
          <span className="px-4 py-2 text-sm">
            Page {currentPage} of {totalPages}
          </span>
          <button
            onClick={() => setCurrentPage(prev => Math.min(totalPages, prev + 1))}
            disabled={currentPage === totalPages}
            className="px-4 py-2 border border-gray-300 rounded-lg disabled:opacity-50 disabled:cursor-not-allowed hover:bg-gray-100"
          >
            Next
          </button>
        </div>
      </div>
    </div>
  );
}
```

**Usage Example:**
```typescript
// pages/Members.tsx
import { PaginatedTable } from '../components/PaginatedTable';

export function Members() {
  const members = [
    { id: 1, name: 'John Doe', email: 'john@example.com', status: 'Active' },
    { id: 2, name: 'Jane Smith', email: 'jane@example.com', status: 'Active' },
  ];

  return (
    <div className="container mx-auto px-4 py-8">
      <h1 className="text-3xl font-bold mb-6">Members</h1>
      <PaginatedTable
        data={members}
        columns={[
          { key: 'name', label: 'Name' },
          { key: 'email', label: 'Email' },
          { 
            key: 'status', 
            label: 'Status',
            render: (value) => (
              <span className={`px-2 py-1 rounded ${
                value === 'Active' ? 'bg-green-100 text-green-800' : 'bg-gray-100 text-gray-800'
              }`}>
                {value}
              </span>
            )
          },
        ]}
        searchable
        onRowClick={(row) => console.log('Clicked:', row)}
      />
    </div>
  );
}
```

---

### Delete Record

**Purpose:** Confirmation dialog for deleting records.

**Design:**
- Modal dialog
- Clear warning message
- Cancel and Delete buttons
- Material Design dialog component

**Template:**

```typescript
// components/DeleteDialog.tsx
import React from 'react';

interface DeleteDialogProps {
  open: boolean;
  title: string;
  message: string;
  itemName?: string;
  onConfirm: () => void;
  onCancel: () => void;
  loading?: boolean;
}

export function DeleteDialog({
  open,
  title,
  message,
  itemName,
  onConfirm,
  onCancel,
  loading = false,
}: DeleteDialogProps) {
  if (!open) return null;

  return (
    <div className="fixed inset-0 z-50 flex items-center justify-center bg-black bg-opacity-50">
      <div className="bg-white rounded-lg shadow-xl max-w-md w-full mx-4">
        <div className="p-6">
          <h2 className="text-xl font-bold text-gray-900 mb-4">
            {title}
          </h2>
          <p className="text-gray-600 mb-6">
            {message}
            {itemName && (
              <span className="font-semibold text-gray-900"> {itemName}</span>
            )}?
          </p>
          <div className="flex justify-end space-x-4">
            <button
              onClick={onCancel}
              disabled={loading}
              className="px-4 py-2 border border-gray-300 rounded-lg text-gray-700 hover:bg-gray-50 disabled:opacity-50"
            >
              Cancel
            </button>
            <button
              onClick={onConfirm}
              disabled={loading}
              className="px-4 py-2 bg-red-600 text-white rounded-lg hover:bg-red-700 disabled:opacity-50"
            >
              {loading ? 'Deleting...' : 'Delete'}
            </button>
          </div>
        </div>
      </div>
    </div>
  );
}
```

**Usage Example:**
```typescript
// Usage in a component
const [deleteDialog, setDeleteDialog] = useState<{
  open: boolean;
  itemId: string | null;
}>({ open: false, itemId: null });

const handleDelete = async (id: string) => {
  setDeleteDialog({ open: true, itemId: id });
};

const confirmDelete = async () => {
  if (deleteDialog.itemId) {
    await deleteRecord(deleteDialog.itemId);
    setDeleteDialog({ open: false, itemId: null });
  }
};

return (
  <>
    <button onClick={() => handleDelete(item.id)}>Delete</button>
    <DeleteDialog
      open={deleteDialog.open}
      title="Delete Member"
      message="Are you sure you want to delete"
      itemName={item.name}
      onConfirm={confirmDelete}
      onCancel={() => setDeleteDialog({ open: false, itemId: null })}
    />
  </>
);
```

---

### Update Record

**Purpose:** Form for updating existing records.

**Design:**
- Form layout with Material Design inputs
- Validation
- Save and Cancel buttons
- Loading states

**Template:**

```typescript
// components/UpdateForm.tsx
import React, { useState, useEffect } from 'react';
import { useNavigate } from 'react-router-dom';

interface UpdateFormProps<T> {
  initialData: T;
  onSubmit: (data: T) => Promise<void>;
  onCancel?: () => void;
  title: string;
  fields: {
    key: keyof T;
    label: string;
    type?: 'text' | 'email' | 'number' | 'date' | 'textarea' | 'select';
    required?: boolean;
    options?: { value: string; label: string }[];
    validate?: (value: any) => string | null;
  }[];
  loading?: boolean;
}

export function UpdateForm<T extends Record<string, any>>({
  initialData,
  onSubmit,
  onCancel,
  title,
  fields,
  loading = false,
}: UpdateFormProps<T>) {
  const [formData, setFormData] = useState<T>(initialData);
  const [errors, setErrors] = useState<Record<string, string>>({});
  const navigate = useNavigate();

  const handleChange = (key: keyof T, value: any) => {
    setFormData(prev => ({ ...prev, [key]: value }));
    // Clear error when user starts typing
    if (errors[String(key)]) {
      setErrors(prev => {
        const newErrors = { ...prev };
        delete newErrors[String(key)];
        return newErrors;
      });
    }
  };

  const validate = (): boolean => {
    const newErrors: Record<string, string> = {};

    fields.forEach(field => {
      const value = formData[field.key];
      
      if (field.required && (!value || value === '')) {
        newErrors[String(field.key)] = `${field.label} is required`;
      } else if (field.validate) {
        const error = field.validate(value);
        if (error) {
          newErrors[String(field.key)] = error;
        }
      }
    });

    setErrors(newErrors);
    return Object.keys(newErrors).length === 0;
  };

  const handleSubmit = async (e: React.FormEvent) => {
    e.preventDefault();
    
    if (!validate()) {
      return;
    }

    try {
      await onSubmit(formData);
      if (onCancel) {
        onCancel();
      } else {
        navigate(-1);
      }
    } catch (error) {
      console.error('Update failed:', error);
    }
  };

  return (
    <div className="bg-white rounded-lg shadow-md p-6">
      <h2 className="text-2xl font-bold mb-6">{title}</h2>
      
      <form onSubmit={handleSubmit}>
        {fields.map(field => (
          <div key={String(field.key)} className="mb-6">
            <label className="block text-sm font-medium text-gray-700 mb-2">
              {field.label}
              {field.required && <span className="text-red-500 ml-1">*</span>}
            </label>
            
            {field.type === 'textarea' ? (
              <textarea
                value={formData[field.key] || ''}
                onChange={(e) => handleChange(field.key, e.target.value)}
                className={`w-full px-4 py-2 border rounded-lg focus:outline-none focus:ring-2 focus:ring-[#000033] ${
                  errors[String(field.key)] ? 'border-red-500' : 'border-gray-300'
                }`}
                rows={4}
              />
            ) : field.type === 'select' ? (
              <select
                value={formData[field.key] || ''}
                onChange={(e) => handleChange(field.key, e.target.value)}
                className={`w-full px-4 py-2 border rounded-lg focus:outline-none focus:ring-2 focus:ring-[#000033] ${
                  errors[String(field.key)] ? 'border-red-500' : 'border-gray-300'
                }`}
              >
                <option value="">Select {field.label}</option>
                {field.options?.map(option => (
                  <option key={option.value} value={option.value}>
                    {option.label}
                  </option>
                ))}
              </select>
            ) : (
              <input
                type={field.type || 'text'}
                value={formData[field.key] || ''}
                onChange={(e) => handleChange(field.key, e.target.value)}
                className={`w-full px-4 py-2 border rounded-lg focus:outline-none focus:ring-2 focus:ring-[#000033] ${
                  errors[String(field.key)] ? 'border-red-500' : 'border-gray-300'
                }`}
              />
            )}
            
            {errors[String(field.key)] && (
              <p className="mt-1 text-sm text-red-500">
                {errors[String(field.key)]}
              </p>
            )}
          </div>
        ))}

        <div className="flex justify-end space-x-4 mt-8">
          <button
            type="button"
            onClick={onCancel || (() => navigate(-1))}
            disabled={loading}
            className="px-6 py-2 border border-gray-300 rounded-lg text-gray-700 hover:bg-gray-50 disabled:opacity-50"
          >
            Cancel
          </button>
          <button
            type="submit"
            disabled={loading}
            className="px-6 py-2 bg-[#000033] text-white rounded-lg hover:opacity-90 disabled:opacity-50"
          >
            {loading ? 'Saving...' : 'Save Changes'}
          </button>
        </div>
      </form>
    </div>
  );
}
```

**Usage Example:**
```typescript
// pages/EditMember.tsx
import { UpdateForm } from '../components/UpdateForm';

export function EditMember() {
  const member = { id: 1, name: 'John Doe', email: 'john@example.com' };

  const handleSubmit = async (data: typeof member) => {
    await updateMember(member.id, data);
  };

  return (
    <div className="container mx-auto px-4 py-8">
      <UpdateForm
        initialData={member}
        onSubmit={handleSubmit}
        title="Edit Member"
        fields={[
          { key: 'name', label: 'Name', type: 'text', required: true },
          { 
            key: 'email', 
            label: 'Email', 
            type: 'email', 
            required: true,
            validate: (value) => {
              if (!/^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(value)) {
                return 'Invalid email address';
              }
              return null;
            }
          },
        ]}
      />
    </div>
  );
}
```

---

### Insert Record

**Purpose:** Form for creating new records.

**Design:**
- Similar to Update Form but for new records
- Empty initial state
- Create button instead of Save

**Template:**

```typescript
// components/InsertForm.tsx
import React, { useState } from 'react';
import { useNavigate } from 'react-router-dom';

interface InsertFormProps<T> {
  onSubmit: (data: Partial<T>) => Promise<void>;
  onCancel?: () => void;
  title: string;
  fields: {
    key: keyof T;
    label: string;
    type?: 'text' | 'email' | 'number' | 'date' | 'textarea' | 'select';
    required?: boolean;
    defaultValue?: any;
    options?: { value: string; label: string }[];
    validate?: (value: any) => string | null;
  }[];
  loading?: boolean;
}

export function InsertForm<T extends Record<string, any>>({
  onSubmit,
  onCancel,
  title,
  fields,
  loading = false,
}: InsertFormProps<T>) {
  const [formData, setFormData] = useState<Partial<T>>(
    fields.reduce((acc, field) => {
      if (field.defaultValue !== undefined) {
        acc[field.key] = field.defaultValue;
      }
      return acc;
    }, {} as Partial<T>)
  );
  const [errors, setErrors] = useState<Record<string, string>>({});
  const navigate = useNavigate();

  const handleChange = (key: keyof T, value: any) => {
    setFormData(prev => ({ ...prev, [key]: value }));
    if (errors[String(key)]) {
      setErrors(prev => {
        const newErrors = { ...prev };
        delete newErrors[String(key)];
        return newErrors;
      });
    }
  };

  const validate = (): boolean => {
    const newErrors: Record<string, string> = {};

    fields.forEach(field => {
      const value = formData[field.key];
      
      if (field.required && (!value || value === '')) {
        newErrors[String(field.key)] = `${field.label} is required`;
      } else if (field.validate) {
        const error = field.validate(value);
        if (error) {
          newErrors[String(field.key)] = error;
        }
      }
    });

    setErrors(newErrors);
    return Object.keys(newErrors).length === 0;
  };

  const handleSubmit = async (e: React.FormEvent) => {
    e.preventDefault();
    
    if (!validate()) {
      return;
    }

    try {
      await onSubmit(formData);
      if (onCancel) {
        onCancel();
      } else {
        navigate(-1);
      }
    } catch (error) {
      console.error('Create failed:', error);
    }
  };

  return (
    <div className="bg-white rounded-lg shadow-md p-6">
      <h2 className="text-2xl font-bold mb-6">{title}</h2>
      
      <form onSubmit={handleSubmit}>
        {fields.map(field => (
          <div key={String(field.key)} className="mb-6">
            <label className="block text-sm font-medium text-gray-700 mb-2">
              {field.label}
              {field.required && <span className="text-red-500 ml-1">*</span>}
            </label>
            
            {field.type === 'textarea' ? (
              <textarea
                value={formData[field.key] || ''}
                onChange={(e) => handleChange(field.key, e.target.value)}
                className={`w-full px-4 py-2 border rounded-lg focus:outline-none focus:ring-2 focus:ring-[#000033] ${
                  errors[String(field.key)] ? 'border-red-500' : 'border-gray-300'
                }`}
                rows={4}
              />
            ) : field.type === 'select' ? (
              <select
                value={formData[field.key] || ''}
                onChange={(e) => handleChange(field.key, e.target.value)}
                className={`w-full px-4 py-2 border rounded-lg focus:outline-none focus:ring-2 focus:ring-[#000033] ${
                  errors[String(field.key)] ? 'border-red-500' : 'border-gray-300'
                }`}
              >
                <option value="">Select {field.label}</option>
                {field.options?.map(option => (
                  <option key={option.value} value={option.value}>
                    {option.label}
                  </option>
                ))}
              </select>
            ) : (
              <input
                type={field.type || 'text'}
                value={formData[field.key] || ''}
                onChange={(e) => handleChange(field.key, e.target.value)}
                className={`w-full px-4 py-2 border rounded-lg focus:outline-none focus:ring-2 focus:ring-[#000033] ${
                  errors[String(field.key)] ? 'border-red-500' : 'border-gray-300'
                }`}
              />
            )}
            
            {errors[String(field.key)] && (
              <p className="mt-1 text-sm text-red-500">
                {errors[String(field.key)]}
              </p>
            )}
          </div>
        ))}

        <div className="flex justify-end space-x-4 mt-8">
          <button
            type="button"
            onClick={onCancel || (() => navigate(-1))}
            disabled={loading}
            className="px-6 py-2 border border-gray-300 rounded-lg text-gray-700 hover:bg-gray-50 disabled:opacity-50"
          >
            Cancel
          </button>
          <button
            type="submit"
            disabled={loading}
            className="px-6 py-2 bg-[#000033] text-white rounded-lg hover:opacity-90 disabled:opacity-50"
          >
            {loading ? 'Creating...' : 'Create'}
          </button>
        </div>
      </form>
    </div>
  );
}
```

---

### Static Content Page

**Purpose:** Pages for about, help, privacy policy, terms, etc.

**Design:**
- Clean content layout
- Readable typography
- Consistent styling

**Template:**

```typescript
// components/StaticContentPage.tsx
import React from 'react';
import { Header } from './Header';
import { Footer } from './Footer';

interface StaticContentPageProps {
  title: string;
  children: React.ReactNode;
}

export function StaticContentPage({ title, children }: StaticContentPageProps) {
  return (
    <div className="min-h-screen flex flex-col bg-white">
      <Header />
      
      <main className="flex-grow container mx-auto px-4 py-12 max-w-4xl">
        <h1 className="text-4xl font-bold text-gray-900 mb-8">{title}</h1>
        
        <div className="prose prose-lg max-w-none">
          {children}
        </div>
      </main>

      <Footer />
    </div>
  );
}
```

**Usage Example:**
```typescript
// pages/About.tsx
import { StaticContentPage } from '../components/StaticContentPage';

export function About() {
  return (
    <StaticContentPage title="About Cogento">
      <p>
        Cogento is a self-hosted membership management system designed for
        small clubs and non-profits.
      </p>
      
      <h2>Our Mission</h2>
      <p>
        To provide a free, flexible, and powerful solution for managing
        members, subscriptions, and payments.
      </p>
      
      <h2>Key Features</h2>
      <ul>
        <li>Stripe as source of truth</li>
        <li>Self-hosted deployment</li>
        <li>Multi-tenant support</li>
      </ul>
    </StaticContentPage>
  );
}
```

---

## Layout Patterns

### Standard Page Layout

```typescript
// layouts/StandardLayout.tsx
import React from 'react';
import { Header } from '../components/Header';
import { Footer } from '../components/Footer';

interface StandardLayoutProps {
  children: React.ReactNode;
}

export function StandardLayout({ children }: StandardLayoutProps) {
  return (
    <div className="min-h-screen flex flex-col bg-white">
      <Header />
      <main className="flex-grow container mx-auto px-4 py-8">
        {children}
      </main>
      <Footer />
    </div>
  );
}
```

---

## Responsive Design

### Breakpoints

```css
/* Tailwind CSS breakpoints */
sm: 640px   /* Small devices */
md: 768px   /* Medium devices */
lg: 1024px  /* Large devices */
xl: 1280px  /* Extra large devices */
2xl: 1536px /* 2X Extra large devices */
```

### Mobile-First Approach

- Design for mobile first
- Use responsive Tailwind classes
- Test on multiple screen sizes
- Ensure touch targets are at least 44x44px

---

## Summary

**UI Standards:**
- ✅ Edgible color scheme: `#000033` for headers/footers, white for pages
- ✅ Material Design components for consistency
- ✅ Tailwind CSS for styling
- ✅ Responsive design
- ✅ Reusable component templates

**Component Templates:**
1. ✅ Header - Fixed navigation with Edgible branding
2. ✅ Footer - Site-wide footer with links
3. ✅ Landing Page - Initial page with hero section
4. ✅ Paginated Table - Data table with pagination
5. ✅ Delete Record - Confirmation dialog
6. ✅ Update Record - Form for editing
7. ✅ Insert Record - Form for creating
8. ✅ Static Content Page - Content pages (about, help, etc.)

**Key Design Principles:**
- Consistency across all pages
- Accessibility
- Responsive design
- Clean, modern interface
- Edgible brand identity
