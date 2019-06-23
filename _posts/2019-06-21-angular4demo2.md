---
layout: post
title: Angular client with Asp.Net Core WebApi backend (Part 2)
categories: repos angular4demo
permalink: /:categories/part2.html
---

## Introduction

I created this solution to help me learn more about Angular and how I could use it to create a modern website.  I decided to write both a front-end SPA that runs in the browser as well as a backend api that would serve up the data through REST endpoints.  I had already done a bit of work with .Net, so I created the backend with Asp.Net Core WebApi.  This post covers the code and design of the **Angular** project.

*NOTE: I actually wrote the code for this repository a while ago, but I thought I'd write this post to document a bit of how it works*

## Overview

* Front-end SPA is created with Angular 4 written in Typescript
* Back-end REST service endpoints are created with Asp.Net Core WebApi
* The data exists in an Entity Framework Core In-memory database.  Random, fake data is generated at startup.
* Security for the service endpoints use Json Web Tokens
* A unit test project to test the WebApi code also exists as part of the solution.
* The database schema has two tables with Customers and Orders.

## The Code

### Components

Angular component classes have been created for each of the pages that are to be displayed.  For managing the customer data, there are three components for the list, edit, and details screens.  

![screenshot]({{ '/assets/angular4demo/images/client-app-components-customer.png' | relative_url }})

The `CustomerListComponent` contains some basic code for making a call to one of the REST endpoints and displaying the data on the screen.  The `CustomerService` service is injected into the constructor and the data is loaded upon initialization by calling a custom `loadData` function inside of the class.  That function then calls a method on the `CustomerService`, which returns an `Observable`.  When we subscribe to the Observable, we include a callback function that sets values on the component class.

```typescript

export class CustomerListComponent implements OnInit {

    data: PagedData<ICustomer>;
    public totalPages: number = 0;
    currentPage: number = 1;
    isLoading: boolean;

    constructor(private _customerService: CustomerService) { }

    ngOnInit(): void {
        this.isLoading = true;
        this.loadData();
    }

    loadData(): void {
        this._customerService.getPaged(this.currentPage - 1, 10).subscribe(result => {
            this.data = result;
            this.currentPage = this.data.pageIndex + 1;
            this.isLoading = false;
        }, error => this.handleError(error));
    }

    // ... some code here

}

```

The UI is bound to the properties on the `CustomerListComponent`.   

```html

<table class='table table-striped table-hover table-condensed x-table-smaller-text'>
    <thead>
        <tr>
            <th>Id</th>
            <th>First Name</th>
            <th>Last Name</th>
            <th>Email</th>
            <th></th>
        </tr>
    </thead>
    <tbody>
        <tr *ngFor="let customer of data.items">
            <td>{{ customer.id }}</td>
            <td>{{ customer.firstName }}</td>
            <td>{{ customer.lastName }}</td>
            <td>{{ customer.email }}</td>
            <td><a [routerLink]="['/customers', customer.id, 'edit']">Edit</a> |
                <a [routerLink]="['/customers', customer.id, 'details']">Details</a></td>
        </tr>
    </tbody>
</table>

```

The rest of the display screens have similar code for the components and html with some additional code where necessary to do things such as submit the data to the server.  I have also written a custom component to display validation messages called `ValidationMessageComponent`.  This works with the standard `Validators` to display the error messages in a consistent manner with some clean markup. 

```html

<div class="form-group row" [ngClass]="{'has-error': mainForm.get('firstName').invalid}">
    <label for="firstName" class="col-sm-2 col-form-label">First Name</label>
    <div class="col-sm-4">
        <input type="text" class="form-control" id="firstName" formControlName="firstName">
        <app-validation-message [for]="mainForm.get('firstName')" propertyName="First Name"></app-validation-message>
    </div>
</div>

```

### Data Entities

I have defined data entity interfaces to pass data back and forth between the client and the REST services.  These data entities correspond to the DTOs defined in the WebApi project.

WebApi DTO:

```csharp

public class CustomerDto
{
    public int Id { get; set; }
    public string FirstName { get; set; }
    public string LastName { get; set; }
    public string Email { get; set; }
}

```

Angular data entity:

```typescript

export interface ICustomer {
    id: number;
    firstName: string;
    lastName: string;
    email: string;
}

```

### Services

Typescript service classes have been written to communicate with the REST services.  They encapsulate all of the functionality we need for the REST services and can be injected into the Angular component classes.

```typescript

@Injectable()
export class CustomerService {

    constructor(private http: HttpClient, private config: AppConfig) { }

    getPaged(pageIndex: number, pageSize: number): Observable<PagedData<ICustomer>> {

        return this.http.get<PagedData<ICustomer>>(this.config.apiBaseUrl + `customers?pageIndex=${pageIndex}&pageSize=${pageSize}`);

    }

    getById(id: number): Observable<ICustomer> {

        return this.http.get<ICustomer>(this.config.apiBaseUrl + 'customers/' + id);

    }

    create(customer: ICustomer): Observable<ICustomer> {

        return this.http.post<ICustomer>(this.config.apiBaseUrl + 'customers', customer);

    }

    update(id: number, customer: ICustomer): Observable<Response> {

        return this.http.put<Response>(this.config.apiBaseUrl + 'customers/' + id, customer);

    }

    delete(id: number): Observable<Response> {

        return this.http.delete<Response>(this.config.apiBaseUrl + 'customers/' + id);

    }
}

```

The `AuthenticationService` communicates with the REST services on the server and adds/removes the security token in local storage.

```typescript

@Injectable()
export class AuthenticationService {
    constructor(private http: HttpClient, private config: AppConfig) { }

    login(username: string, password: string): Observable<boolean> {
        return this.http.post<AuthToken>(this.config.apiBaseUrl + 'token', { userName: username, password: password })
            .do(result => {
                localStorage.setItem(this.config.authTokenName, result.token);
            })
            .map(result => true);
    }

    logout() {
        // remove auth token from local storage to log user out
        localStorage.removeItem(this.config.authTokenName);
    }
}

```

### Infrastructure

I have included some additional code that is part of the infrastructure of the site.  This includes Auth Guards and Http Interceptors.  The `AuthInterceptor` is responsible for getting the security token out of the local storage and adding it to the request header before it is sent to the REST service endpoints.

```typescript

@Injectable()
export class AuthInterceptor implements HttpInterceptor {
    constructor(private config: AppConfig) { }

    intercept(req: HttpRequest<any>, next: HttpHandler): Observable<HttpEvent<any>> {
        // Get the auth header from the service.
        const token = localStorage.getItem(this.config.authTokenName);

        if (token) {
            // Clone the request to add the new header.
            const authReq = req.clone({ headers: req.headers.set('Authorization', `Bearer ${token}`) });
            // Pass on the cloned request instead of the original request.
            return next.handle(authReq);
        } else {
            return next.handle(req);
        }
    }
}

```
There is also a `TimingInterceptor` that logs the elapsed time it takes to return results from the REST service calls.  You can see the output in the console log of the browser.

![screenshot]({{ '/assets/angular4demo/images/http-interceptor-timing-log.png' | relative_url }})

The `app.module.shared.ts` file in the project includes code to include the auth guards and interceptors appropriately.  This is also where all of the page routing is configured.

```typescript

    imports: [
        CommonModule,
        HttpClientModule,
        FormsModule,
        ReactiveFormsModule,
        RouterModule.forRoot([
            { path: '', redirectTo: 'home', pathMatch: 'full' },
            { path: 'login', component: LoginComponent },
            { path: 'home', component: HomeComponent },
            { path: 'customers', component: CustomerListComponent, canActivate: [AuthGuard] },
            { path: 'customers/add', component: CustomerEditComponent, canActivate: [AuthGuard] },
            { path: 'customers/:id/edit', component: CustomerEditComponent, canActivate: [AuthGuard] },
            { path: 'customers/:id/details', component: CustomerDetailsComponent, canActivate: [AuthGuard] },
            { path: '**', redirectTo: 'home' }
        ]),
        PaginationModule.forRoot()
    ],
    providers: [
        CustomerService,
        OrderService,
        AuthenticationService,
        AppConfig,
        AuthGuard,
        { provide: HTTP_INTERCEPTORS, useClass: TimingInterceptor, multi: true },
        { provide: HTTP_INTERCEPTORS, useClass: AuthInterceptor, multi: true }
    ]

```

### Security

The REST services use Json Web Tokens (JWTs) for authorization.  Some of the code for how that is incorporated into the Angular application can be seen above in the `AuthenticationService` and the `AuthInterceptor`.  Additional code is also in the `AuthGuard` class.  If the user tries to access any pages that require authorization, the application uses the `AuthGuard` to check for a valid JWT.  If the token does not exist, the user is redirected to the login page.  Otherwise, they are allowed to proceed to the page as desired.

## The UI

The following images show the basics of the UI that is accomplished through the above code.  A pre-defined template was used for the basic UI and home page, but this could easily be changed and stylized with a custom theme as desired.

### Home Page

![screenshot]({{ '/assets/angular4demo/images/app-home-page.png' | relative_url }})

### Login Page

![screenshot]({{ '/assets/angular4demo/images/app-login-page.png' | relative_url }})

### Customer List Page

![screenshot]({{ '/assets/angular4demo/images/app-customers-page.png' | relative_url }})

### Customer Edit Page

![screenshot]({{ '/assets/angular4demo/images/app-customer-edit-page.png' | relative_url }})

### Customer Details Page

![screenshot]({{ '/assets/angular4demo/images/app-customer-details-page.png' | relative_url }})


## Conclusion

Angular provides a nice framework to create Single Page Applications with html and Typescript.  I had to learn a few new technologies and frameworks to complete this, but I'm pretty happy with the result and how I could continue with these patterns to create a much larger and more complex site with many pages.

## See also

* <https://angular.io/>
* <https://www.typescriptlang.org/>
* <https://rxjs-dev.firebaseapp.com/guide/overview>
