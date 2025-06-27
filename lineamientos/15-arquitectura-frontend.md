# Arquitectura Frontend

## Propósito

La arquitectura frontend define cómo se estructura, desarrolla y despliega la capa de presentación de las aplicaciones. En nuestro contexto, nos enfocamos en React y Angular como frameworks principales, con énfasis en escalabilidad, mantenibilidad y experiencia de usuario.

## Stack Tecnológico Frontend

### Frameworks Principales

#### React (Recomendado para nuevos proyectos)
```json
{
  "framework": "React 18+",
  "language": "TypeScript",
  "buildTool": "Vite",
  "stateManagement": "Redux Toolkit / Zustand",
  "styling": "Tailwind CSS / Styled Components",
  "testing": "Jest + React Testing Library",
  "routing": "React Router v6",
  "forms": "React Hook Form",
  "http": "Axios / React Query"
}
```

#### Angular (Para proyectos empresariales complejos)
```json
{
  "framework": "Angular 17+",
  "language": "TypeScript",
  "buildTool": "Angular CLI",
  "stateManagement": "NgRx / RxJS",
  "styling": "Angular Material / Tailwind CSS",
  "testing": "Jasmine + Karma",
  "routing": "Angular Router",
  "forms": "Reactive Forms",
  "http": "Angular HttpClient"
}
```

## Arquitectura de Componentes

### Patrón de Componentes React
```typescript
// components/common/Button/Button.tsx
import React from 'react';
import { ButtonProps } from './Button.types';
import { StyledButton } from './Button.styles';

export const Button: React.FC<ButtonProps> = ({
  children,
  variant = 'primary',
  size = 'medium',
  disabled = false,
  onClick,
  type = 'button',
  ...props
}) => {
  return (
    <StyledButton
      variant={variant}
      size={size}
      disabled={disabled}
      onClick={onClick}
      type={type}
      {...props}
    >
      {children}
    </StyledButton>
  );
};

// components/common/Button/Button.types.ts
export interface ButtonProps {
  children: React.ReactNode;
  variant?: 'primary' | 'secondary' | 'danger' | 'ghost';
  size?: 'small' | 'medium' | 'large';
  disabled?: boolean;
  onClick?: (event: React.MouseEvent<HTMLButtonElement>) => void;
  type?: 'button' | 'submit' | 'reset';
  className?: string;
}

// components/common/Button/Button.styles.ts
import styled, { css } from 'styled-components';

const buttonVariants = {
  primary: css`
    background-color: #007bff;
    color: white;
    &:hover:not(:disabled) {
      background-color: #0056b3;
    }
  `,
  secondary: css`
    background-color: #6c757d;
    color: white;
    &:hover:not(:disabled) {
      background-color: #545b62;
    }
  `,
  danger: css`
    background-color: #dc3545;
    color: white;
    &:hover:not(:disabled) {
      background-color: #c82333;
    }
  `,
  ghost: css`
    background-color: transparent;
    color: #007bff;
    border: 1px solid #007bff;
    &:hover:not(:disabled) {
      background-color: #007bff;
      color: white;
    }
  `
};

const buttonSizes = {
  small: css`
    padding: 0.375rem 0.75rem;
    font-size: 0.875rem;
  `,
  medium: css`
    padding: 0.5rem 1rem;
    font-size: 1rem;
  `,
  large: css`
    padding: 0.75rem 1.5rem;
    font-size: 1.125rem;
  `
};

export const StyledButton = styled.button<ButtonProps>`
  border: none;
  border-radius: 0.375rem;
  font-weight: 500;
  cursor: pointer;
  transition: all 0.2s ease-in-out;

  ${({ variant }) => buttonVariants[variant || 'primary']}
  ${({ size }) => buttonSizes[size || 'medium']}

  &:disabled {
    opacity: 0.6;
    cursor: not-allowed;
  }

  &:focus {
    outline: 2px solid #007bff;
    outline-offset: 2px;
  }
`;
```

### Patrón de Componentes Angular
```typescript
// components/common/button/button.component.ts
import { Component, Input, Output, EventEmitter } from '@angular/core';
import { CommonModule } from '@angular/common';

export type ButtonVariant = 'primary' | 'secondary' | 'danger' | 'ghost';
export type ButtonSize = 'small' | 'medium' | 'large';

@Component({
  selector: 'app-button',
  standalone: true,
  imports: [CommonModule],
  template: `
    <button
      [class]="buttonClasses"
      [disabled]="disabled"
      (click)="onClick.emit($event)"
      [type]="type"
    >
      <ng-content></ng-content>
    </button>
  `,
  styles: [`
    button {
      border: none;
      border-radius: 0.375rem;
      font-weight: 500;
      cursor: pointer;
      transition: all 0.2s ease-in-out;
    }

    button:disabled {
      opacity: 0.6;
      cursor: not-allowed;
    }

    button:focus {
      outline: 2px solid #007bff;
      outline-offset: 2px;
    }

    .btn-primary {
      background-color: #007bff;
      color: white;
    }

    .btn-primary:hover:not(:disabled) {
      background-color: #0056b3;
    }

    .btn-secondary {
      background-color: #6c757d;
      color: white;
    }

    .btn-secondary:hover:not(:disabled) {
      background-color: #545b62;
    }

    .btn-danger {
      background-color: #dc3545;
      color: white;
    }

    .btn-danger:hover:not(:disabled) {
      background-color: #c82333;
    }

    .btn-ghost {
      background-color: transparent;
      color: #007bff;
      border: 1px solid #007bff;
    }

    .btn-ghost:hover:not(:disabled) {
      background-color: #007bff;
      color: white;
    }

    .btn-small {
      padding: 0.375rem 0.75rem;
      font-size: 0.875rem;
    }

    .btn-medium {
      padding: 0.5rem 1rem;
      font-size: 1rem;
    }

    .btn-large {
      padding: 0.75rem 1.5rem;
      font-size: 1.125rem;
    }
  `]
})
export class ButtonComponent {
  @Input() variant: ButtonVariant = 'primary';
  @Input() size: ButtonSize = 'medium';
  @Input() disabled = false;
  @Input() type: 'button' | 'submit' | 'reset' = 'button';
  @Output() onClick = new EventEmitter<MouseEvent>();

  get buttonClasses(): string {
    return `btn-${this.variant} btn-${this.size}`;
  }
}
```

## Gestión de Estado

### Redux Toolkit (React)
```typescript
// store/slices/authSlice.ts
import { createSlice, createAsyncThunk, PayloadAction } from '@reduxjs/toolkit';
import { AuthService } from '../../services/authService';

export interface User {
  id: string;
  email: string;
  name: string;
  role: string;
}

export interface AuthState {
  user: User | null;
  token: string | null;
  isLoading: boolean;
  error: string | null;
}

const initialState: AuthState = {
  user: null,
  token: localStorage.getItem('token'),
  isLoading: false,
  error: null
};

export const login = createAsyncThunk(
  'auth/login',
  async (credentials: { email: string; password: string }) => {
    const response = await AuthService.login(credentials);
    localStorage.setItem('token', response.token);
    return response;
  }
);

export const logout = createAsyncThunk(
  'auth/logout',
  async () => {
    await AuthService.logout();
    localStorage.removeItem('token');
  }
);

const authSlice = createSlice({
  name: 'auth',
  initialState,
  reducers: {
    clearError: (state) => {
      state.error = null;
    }
  },
  extraReducers: (builder) => {
    builder
      .addCase(login.pending, (state) => {
        state.isLoading = true;
        state.error = null;
      })
      .addCase(login.fulfilled, (state, action) => {
        state.isLoading = false;
        state.user = action.payload.user;
        state.token = action.payload.token;
      })
      .addCase(login.rejected, (state, action) => {
        state.isLoading = false;
        state.error = action.error.message || 'Login failed';
      })
      .addCase(logout.fulfilled, (state) => {
        state.user = null;
        state.token = null;
      });
  }
});

export const { clearError } = authSlice.actions;
export default authSlice.reducer;
```

### NgRx (Angular)
```typescript
// store/auth/auth.actions.ts
import { createAction, props } from '@ngrx/store';

export const login = createAction(
  '[Auth] Login',
  props<{ email: string; password: string }>()
);

export const loginSuccess = createAction(
  '[Auth] Login Success',
  props<{ user: any; token: string }>()
);

export const loginFailure = createAction(
  '[Auth] Login Failure',
  props<{ error: string }>()
);

export const logout = createAction('[Auth] Logout');

export const logoutSuccess = createAction('[Auth] Logout Success');

// store/auth/auth.reducer.ts
import { createReducer, on } from '@ngrx/store';
import * as AuthActions from './auth.actions';

export interface AuthState {
  user: any;
  token: string | null;
  isLoading: boolean;
  error: string | null;
}

export const initialState: AuthState = {
  user: null,
  token: localStorage.getItem('token'),
  isLoading: false,
  error: null
};

export const authReducer = createReducer(
  initialState,
  on(AuthActions.login, (state) => ({
    ...state,
    isLoading: true,
    error: null
  })),
  on(AuthActions.loginSuccess, (state, { user, token }) => ({
    ...state,
    user,
    token,
    isLoading: false
  })),
  on(AuthActions.loginFailure, (state, { error }) => ({
    ...state,
    error,
    isLoading: false
  })),
  on(AuthActions.logoutSuccess, (state) => ({
    ...state,
    user: null,
    token: null
  }))
);
```

## Gestión de Datos y APIs

### React Query (TanStack Query)
```typescript
// hooks/useUsers.ts
import { useQuery, useMutation, useQueryClient } from '@tanstack/react-query';
import { UserService } from '../services/userService';

export const useUsers = () => {
  return useQuery({
    queryKey: ['users'],
    queryFn: UserService.getUsers,
    staleTime: 5 * 60 * 1000, // 5 minutes
    cacheTime: 10 * 60 * 1000, // 10 minutes
  });
};

export const useUser = (id: string) => {
  return useQuery({
    queryKey: ['users', id],
    queryFn: () => UserService.getUser(id),
    enabled: !!id,
  });
};

export const useCreateUser = () => {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: UserService.createUser,
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: ['users'] });
    },
  });
};

export const useUpdateUser = () => {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: ({ id, data }: { id: string; data: Partial<User> }) =>
      UserService.updateUser(id, data),
    onSuccess: (updatedUser) => {
      queryClient.setQueryData(['users', updatedUser.id], updatedUser);
      queryClient.invalidateQueries({ queryKey: ['users'] });
    },
  });
};

// services/userService.ts
import { apiClient } from './apiClient';

export interface User {
  id: string;
  name: string;
  email: string;
  role: string;
}

export class UserService {
  static async getUsers(): Promise<User[]> {
    const response = await apiClient.get('/users');
    return response.data;
  }

  static async getUser(id: string): Promise<User> {
    const response = await apiClient.get(`/users/${id}`);
    return response.data;
  }

  static async createUser(userData: Omit<User, 'id'>): Promise<User> {
    const response = await apiClient.post('/users', userData);
    return response.data;
  }

  static async updateUser(id: string, userData: Partial<User>): Promise<User> {
    const response = await apiClient.put(`/users/${id}`, userData);
    return response.data;
  }

  static async deleteUser(id: string): Promise<void> {
    await apiClient.delete(`/users/${id}`);
  }
}
```

### Angular HttpClient con Interceptors
```typescript
// services/api.service.ts
import { Injectable } from '@angular/core';
import { HttpClient, HttpHeaders } from '@angular/common/http';
import { Observable } from 'rxjs';
import { environment } from '../../environments/environment';

@Injectable({
  providedIn: 'root'
})
export class ApiService {
  private baseUrl = environment.apiUrl;

  constructor(private http: HttpClient) {}

  get<T>(endpoint: string): Observable<T> {
    return this.http.get<T>(`${this.baseUrl}${endpoint}`);
  }

  post<T>(endpoint: string, data: any): Observable<T> {
    return this.http.post<T>(`${this.baseUrl}${endpoint}`, data);
  }

  put<T>(endpoint: string, data: any): Observable<T> {
    return this.http.put<T>(`${this.baseUrl}${endpoint}`, data);
  }

  delete<T>(endpoint: string): Observable<T> {
    return this.http.delete<T>(`${this.baseUrl}${endpoint}`);
  }
}

// interceptors/auth.interceptor.ts
import { Injectable } from '@angular/core';
import {
  HttpRequest,
  HttpHandler,
  HttpEvent,
  HttpInterceptor,
  HttpErrorResponse
} from '@angular/common/http';
import { Observable, throwError } from 'rxjs';
import { catchError } from 'rxjs/operators';
import { Router } from '@angular/router';
import { AuthService } from '../services/auth.service';

@Injectable()
export class AuthInterceptor implements HttpInterceptor {
  constructor(
    private authService: AuthService,
    private router: Router
  ) {}

  intercept(request: HttpRequest<unknown>, next: HttpHandler): Observable<HttpEvent<unknown>> {
    const token = this.authService.getToken();

    if (token) {
      request = request.clone({
        setHeaders: {
          Authorization: `Bearer ${token}`
        }
      });
    }

    return next.handle(request).pipe(
      catchError((error: HttpErrorResponse) => {
        if (error.status === 401) {
          this.authService.logout();
          this.router.navigate(['/login']);
        }
        return throwError(() => error);
      })
    );
  }
}
```

## Seguridad Frontend

### Protección contra XSS
```typescript
// utils/sanitizer.ts
import DOMPurify from 'dompurify';

export class Sanitizer {
  static sanitizeHtml(html: string): string {
    return DOMPurify.sanitize(html, {
      ALLOWED_TAGS: ['b', 'i', 'em', 'strong', 'a', 'p', 'br'],
      ALLOWED_ATTR: ['href', 'target']
    });
  }

  static sanitizeUrl(url: string): string {
    return DOMPurify.sanitize(url, { ALLOWED_URI_REGEXP: /^(?:(?:(?:f|ht)tps?|mailto|tel|callto|cid|xmpp):|[^a-z]|[a-z+.\-]+(?:[^a-z+.\-:]|$))/i });
  }

  static escapeHtml(text: string): string {
    const div = document.createElement('div');
    div.textContent = text;
    return div.innerHTML;
  }
}

// React Component con sanitización
const SafeHtml: React.FC<{ html: string }> = ({ html }) => {
  const sanitizedHtml = Sanitizer.sanitizeHtml(html);

  return (
    <div
      dangerouslySetInnerHTML={{ __html: sanitizedHtml }}
      className="safe-html-content"
    />
  );
};
```

### Protección CSRF
```typescript
// services/apiClient.ts
import axios from 'axios';

const apiClient = axios.create({
  baseURL: process.env.REACT_APP_API_URL,
  headers: {
    'Content-Type': 'application/json',
  },
});

// Interceptor para agregar CSRF token
apiClient.interceptors.request.use((config) => {
  const csrfToken = document.querySelector('meta[name="csrf-token"]')?.getAttribute('content');

  if (csrfToken) {
    config.headers['X-CSRF-Token'] = csrfToken;
  }

  return config;
});

// Interceptor para manejar errores
apiClient.interceptors.response.use(
  (response) => response,
  (error) => {
    if (error.response?.status === 403) {
      // CSRF token inválido o expirado
      window.location.reload();
    }
    return Promise.reject(error);
  }
);

export default apiClient;
```

### Content Security Policy
```html
<!-- index.html -->
<!DOCTYPE html>
<html lang="es">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <meta http-equiv="Content-Security-Policy"
        content="default-src 'self';
                 script-src 'self' 'unsafe-inline' 'unsafe-eval' https://cdn.jsdelivr.net;
                 style-src 'self' 'unsafe-inline' https://fonts.googleapis.com;
                 font-src 'self' https://fonts.gstatic.com;
                 img-src 'self' data: https:;
                 connect-src 'self' https://api.example.com;">
  <title>Mi Aplicación</title>
</head>
<body>
  <div id="root"></div>
</body>
</html>
```

## Performance y Optimización

### Code Splitting (React)
```typescript
// App.tsx
import React, { Suspense, lazy } from 'react';
import { BrowserRouter as Router, Routes, Route } from 'react-router-dom';
import LoadingSpinner from './components/LoadingSpinner';

// Lazy loading de componentes
const Dashboard = lazy(() => import('./pages/Dashboard'));
const Users = lazy(() => import('./pages/Users'));
const Settings = lazy(() => import('./pages/Settings'));

const App: React.FC = () => {
  return (
    <Router>
      <Suspense fallback={<LoadingSpinner />}>
        <Routes>
          <Route path="/" element={<Dashboard />} />
          <Route path="/users" element={<Users />} />
          <Route path="/settings" element={<Settings />} />
        </Routes>
      </Suspense>
    </Router>
  );
};

export default App;
```

### Lazy Loading (Angular)
```typescript
// app-routing.module.ts
import { NgModule } from '@angular/core';
import { RouterModule, Routes } from '@angular/router';

const routes: Routes = [
  {
    path: 'dashboard',
    loadChildren: () => import('./pages/dashboard/dashboard.module').then(m => m.DashboardModule)
  },
  {
    path: 'users',
    loadChildren: () => import('./pages/users/users.module').then(m => m.UsersModule)
  },
  {
    path: 'settings',
    loadChildren: () => import('./pages/settings/settings.module').then(m => m.SettingsModule)
  },
  {
    path: '',
    redirectTo: '/dashboard',
    pathMatch: 'full'
  }
];

@NgModule({
  imports: [RouterModule.forRoot(routes)],
  exports: [RouterModule]
})
export class AppRoutingModule { }
```

### Memoización y Optimización
```typescript
// React - useMemo y useCallback
import React, { useMemo, useCallback } from 'react';

interface UserListProps {
  users: User[];
  searchTerm: string;
  onUserSelect: (user: User) => void;
}

const UserList: React.FC<UserListProps> = ({ users, searchTerm, onUserSelect }) => {
  // Memoizar filtrado de usuarios
  const filteredUsers = useMemo(() => {
    return users.filter(user =>
      user.name.toLowerCase().includes(searchTerm.toLowerCase()) ||
      user.email.toLowerCase().includes(searchTerm.toLowerCase())
    );
  }, [users, searchTerm]);

  // Memoizar función de callback
  const handleUserSelect = useCallback((user: User) => {
    onUserSelect(user);
  }, [onUserSelect]);

  return (
    <div className="user-list">
      {filteredUsers.map(user => (
        <UserCard
          key={user.id}
          user={user}
          onSelect={handleUserSelect}
        />
      ))}
    </div>
  );
};

// Angular - OnPush Strategy
@Component({
  selector: 'app-user-list',
  template: `
    <div class="user-list">
      <app-user-card
        *ngFor="let user of filteredUsers; trackBy: trackByUserId"
        [user]="user"
        (select)="onUserSelect($event)"
      ></app-user-card>
    </div>
  `,
  changeDetection: ChangeDetectionStrategy.OnPush
})
export class UserListComponent {
  @Input() users: User[] = [];
  @Input() searchTerm = '';
  @Output() userSelect = new EventEmitter<User>();

  get filteredUsers(): User[] {
    return this.users.filter(user =>
      user.name.toLowerCase().includes(this.searchTerm.toLowerCase()) ||
      user.email.toLowerCase().includes(this.searchTerm.toLowerCase())
    );
  }

  trackByUserId(index: number, user: User): string {
    return user.id;
  }

  onUserSelect(user: User): void {
    this.userSelect.emit(user);
  }
}
```

## Testing

### Testing React Components
```typescript
// components/Button.test.tsx
import React from 'react';
import { render, screen, fireEvent } from '@testing-library/react';
import { Button } from './Button';

describe('Button Component', () => {
  it('renders with correct text', () => {
    render(<Button>Click me</Button>);
    expect(screen.getByText('Click me')).toBeInTheDocument();
  });

  it('calls onClick when clicked', () => {
    const handleClick = jest.fn();
    render(<Button onClick={handleClick}>Click me</Button>);

    fireEvent.click(screen.getByText('Click me'));
    expect(handleClick).toHaveBeenCalledTimes(1);
  });

  it('is disabled when disabled prop is true', () => {
    render(<Button disabled>Click me</Button>);
    expect(screen.getByText('Click me')).toBeDisabled();
  });

  it('applies correct variant classes', () => {
    render(<Button variant="danger">Delete</Button>);
    const button = screen.getByText('Delete');
    expect(button).toHaveClass('btn-danger');
  });
});
```

### Testing Angular Components
```typescript
// components/button/button.component.spec.ts
import { ComponentFixture, TestBed } from '@angular/core/testing';
import { ButtonComponent } from './button.component';

describe('ButtonComponent', () => {
  let component: ButtonComponent;
  let fixture: ComponentFixture<ButtonComponent>;

  beforeEach(async () => {
    await TestBed.configureTestingModule({
      imports: [ButtonComponent]
    }).compileComponents();

    fixture = TestBed.createComponent(ButtonComponent);
    component = fixture.componentInstance;
    fixture.detectChanges();
  });

  it('should create', () => {
    expect(component).toBeTruthy();
  });

  it('should emit click event when clicked', () => {
    spyOn(component.onClick, 'emit');
    const button = fixture.nativeElement.querySelector('button');

    button.click();

    expect(component.onClick.emit).toHaveBeenCalled();
  });

  it('should be disabled when disabled input is true', () => {
    component.disabled = true;
    fixture.detectChanges();

    const button = fixture.nativeElement.querySelector('button');
    expect(button.disabled).toBe(true);
  });

  it('should apply correct CSS classes based on variant and size', () => {
    component.variant = 'danger';
    component.size = 'large';
    fixture.detectChanges();

    const button = fixture.nativeElement.querySelector('button');
    expect(button.classList.contains('btn-danger')).toBe(true);
    expect(button.classList.contains('btn-large')).toBe(true);
  });
});
```

## Accesibilidad (A11y)

### React Accessibility
```typescript
// components/AccessibleButton.tsx
import React from 'react';

interface AccessibleButtonProps {
  children: React.ReactNode;
  onClick: () => void;
  ariaLabel?: string;
  ariaDescribedBy?: string;
  disabled?: boolean;
}

export const AccessibleButton: React.FC<AccessibleButtonProps> = ({
  children,
  onClick,
  ariaLabel,
  ariaDescribedBy,
  disabled = false
}) => {
  return (
    <button
      onClick={onClick}
      disabled={disabled}
      aria-label={ariaLabel}
      aria-describedby={ariaDescribedBy}
      aria-disabled={disabled}
    >
      {children}
    </button>
  );
};

// components/SkipLink.tsx
export const SkipLink: React.FC = () => {
  return (
    <a
      href="#main-content"
      className="skip-link"
      style={{
        position: 'absolute',
        top: '-40px',
        left: '6px',
        background: '#000',
        color: '#fff',
        padding: '8px',
        textDecoration: 'none',
        zIndex: 1000,
        ':focus': {
          top: '6px'
        }
      }}
    >
      Skip to main content
    </a>
  );
};
```

### Angular Accessibility
```typescript
// directives/focus-trap.directive.ts
import { Directive, ElementRef, Input } from '@angular/core';

@Directive({
  selector: '[appFocusTrap]',
  standalone: true
})
export class FocusTrapDirective {
  @Input() appFocusTrap = true;

  constructor(private el: ElementRef) {}

  ngAfterViewInit() {
    if (this.appFocusTrap) {
      this.setupFocusTrap();
    }
  }

  private setupFocusTrap() {
    const focusableElements = this.el.nativeElement.querySelectorAll(
      'button, [href], input, select, textarea, [tabindex]:not([tabindex="-1"])'
    );

    const firstElement = focusableElements[0];
    const lastElement = focusableElements[focusableElements.length - 1];

    this.el.nativeElement.addEventListener('keydown', (e: KeyboardEvent) => {
      if (e.key === 'Tab') {
        if (e.shiftKey) {
          if (document.activeElement === firstElement) {
            e.preventDefault();
            lastElement.focus();
          }
        } else {
          if (document.activeElement === lastElement) {
            e.preventDefault();
            firstElement.focus();
          }
        }
      }
    });
  }
}
```

## Internacionalización (i18n)

### React i18n
```typescript
// i18n/config.ts
import i18n from 'i18next';
import { initReactI18next } from 'react-i18next';

const resources = {
  es: {
    translation: {
      'welcome': 'Bienvenido',
      'login': 'Iniciar sesión',
      'logout': 'Cerrar sesión',
      'email': 'Correo electrónico',
      'password': 'Contraseña',
      'submit': 'Enviar',
      'cancel': 'Cancelar',
      'error': {
        'required': 'Este campo es requerido',
        'invalid_email': 'Correo electrónico inválido',
        'min_length': 'Mínimo {{min}} caracteres'
      }
    }
  },
  en: {
    translation: {
      'welcome': 'Welcome',
      'login': 'Login',
      'logout': 'Logout',
      'email': 'Email',
      'password': 'Password',
      'submit': 'Submit',
      'cancel': 'Cancel',
      'error': {
        'required': 'This field is required',
        'invalid_email': 'Invalid email',
        'min_length': 'Minimum {{min}} characters'
      }
    }
  }
};

i18n
  .use(initReactI18next)
  .init({
    resources,
    lng: 'es',
    fallbackLng: 'es',
    interpolation: {
      escapeValue: false
    }
  });

export default i18n;

// components/LocalizedButton.tsx
import React from 'react';
import { useTranslation } from 'react-i18next';

interface LocalizedButtonProps {
  translationKey: string;
  onClick: () => void;
  variant?: string;
}

export const LocalizedButton: React.FC<LocalizedButtonProps> = ({
  translationKey,
  onClick,
  variant = 'primary'
}) => {
  const { t } = useTranslation();

  return (
    <Button variant={variant} onClick={onClick}>
      {t(translationKey)}
    </Button>
  );
};
```

### Angular i18n
```typescript
// app.module.ts
import { NgModule } from '@angular/core';
import { HttpClient, HttpClientModule } from '@angular/common/http';
import { TranslateModule, TranslateLoader } from '@ngx-translate/core';
import { TranslateHttpLoader } from '@ngx-translate/http-loader';

export function HttpLoaderFactory(http: HttpClient) {
  return new TranslateHttpLoader(http, './assets/i18n/', '.json');
}

@NgModule({
  imports: [
    HttpClientModule,
    TranslateModule.forRoot({
      loader: {
        provide: TranslateLoader,
        useFactory: HttpLoaderFactory,
        deps: [HttpClient]
      },
      defaultLanguage: 'es'
    })
  ]
})
export class AppModule { }

// components/localized-button.component.ts
import { Component, Input, Output, EventEmitter } from '@angular/core';
import { TranslateService } from '@ngx-translate/core';

@Component({
  selector: 'app-localized-button',
  template: `
    <button [class]="buttonClass" (click)="onClick.emit()">
      {{ translationKey | translate }}
    </button>
  `
})
export class LocalizedButtonComponent {
  @Input() translationKey = '';
  @Input() variant = 'primary';
  @Output() onClick = new EventEmitter<void>();

  get buttonClass(): string {
    return `btn-${this.variant}`;
  }
}
```

## Mejores Prácticas

### 1. Arquitectura de Componentes
- Usar composición sobre herencia
- Mantener componentes pequeños y enfocados
- Implementar prop drilling mínimo
- Usar context o state management para datos globales

### 2. Performance
- Implementar lazy loading para rutas y componentes
- Usar memoización apropiadamente
- Optimizar re-renders con React.memo o OnPush strategy
- Implementar virtualización para listas grandes

### 3. Seguridad
- Sanitizar siempre el contenido HTML
- Implementar CSP headers
- Validar inputs del usuario
- Usar HTTPS en producción

### 4. Accesibilidad
- Usar elementos semánticos HTML
- Implementar navegación por teclado
- Proporcionar textos alternativos
- Mantener contraste de colores adecuado

### 5. Testing
- Escribir tests unitarios para componentes
- Implementar tests de integración
- Usar testing library para tests de usuario
- Mantener cobertura de código alta

### 6. Mantenibilidad
- Usar TypeScript para type safety
- Implementar linting y formatting
- Documentar componentes complejos
- Seguir convenciones de naming

## Herramientas Recomendadas

### Desarrollo
- **Vite**: Build tool rápido para React
- **Angular CLI**: CLI oficial de Angular
- **TypeScript**: Type safety
- **ESLint/Prettier**: Linting y formatting

### Testing
- **Jest**: Framework de testing
- **React Testing Library**: Testing de componentes React
- **Jasmine/Karma**: Testing de Angular
- **Cypress**: Testing E2E

### State Management
- **Redux Toolkit**: Para React
- **NgRx**: Para Angular
- **Zustand**: State management ligero
- **RxJS**: Programación reactiva

### Styling
- **Tailwind CSS**: Utility-first CSS
- **Styled Components**: CSS-in-JS para React
- **Angular Material**: Componentes para Angular
- **Sass/SCSS**: Preprocesador CSS

### Build y Deploy
- **Webpack**: Bundling
- **Vite**: Build tool moderno
- **Docker**: Containerización
- **Netlify/Vercel**: Deploy frontend
