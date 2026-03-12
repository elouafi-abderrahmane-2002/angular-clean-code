# ✅ Angular Clean Code — SOLID en pratique

SOLID, en cours, c'est 5 acronymes abstraits. Sur un vrai codebase Angular,
c'est la différence entre un service de 400 lignes qui fait tout et
4 services de 80 lignes qui font chacun une chose bien. Ce projet documente
comment appliquer chaque principe concrètement en Angular, avec des exemples
réels — pas des Counter ou TodoList.

---

## S — Single Responsibility Principle

Un service = une responsabilité. Exemple typique qui viole SRP :

```typescript
// ❌ MAUVAIS : UserService fait trop de choses
@Injectable({ providedIn: 'root' })
export class UserService {
  getUsers() { /* appel HTTP */ }
  formatUserName(user: User): string { /* logique de formatage */ }
  saveToLocalStorage(user: User) { /* persistance */ }
  sendWelcomeEmail(user: User) { /* notification */ }
}
```

```typescript
// ✅ BON : chaque service a une seule raison de changer
@Injectable({ providedIn: 'root' })
export class UserApiService {
  getUsers(): Observable<User[]> { /* appel HTTP uniquement */ }
}

@Injectable({ providedIn: 'root' })
export class UserFormatterService {
  formatName(user: User): string {
    return `${user.firstName} ${user.lastName}`.trim();
  }
}

@Injectable({ providedIn: 'root' })
export class UserStorageService {
  save(user: User): void { localStorage.setItem('user', JSON.stringify(user)); }
  load(): User | null { /* ... */ }
}
```

---

## O — Open/Closed Principle

Ouvert à l'extension, fermé à la modification.
En Angular, c'est l'injection de dépendances + interfaces.

```typescript
// Interface définie dans le Core (ne dépend de rien)
export abstract class NotificationService {
  abstract notify(message: string): void;
}

// Implémentation toast — en production
@Injectable()
export class ToastNotificationService implements NotificationService {
  notify(message: string): void {
    // Afficher un toast
  }
}

// Implémentation console — en tests
@Injectable()
export class ConsoleNotificationService implements NotificationService {
  notify(message: string): void {
    console.log(`[TEST NOTIFICATION] ${message}`);
  }
}

// Dans AppModule / TestModule :
// { provide: NotificationService, useClass: ToastNotificationService }
// { provide: NotificationService, useClass: ConsoleNotificationService }
// → zéro modification du code qui utilise NotificationService
```

---

## Interceptors — séparation des responsabilités HTTP

```typescript
// Un interceptor = une responsabilité

// 1. JWT Interceptor — injecte le token
export class JwtInterceptor implements HttpInterceptor {
  intercept(req: HttpRequest<any>, next: HttpHandler) {
    const token = this.auth.getToken();
    return next.handle(token ? this.addBearer(req, token) : req);
  }
}

// 2. Error Interceptor — gestion centralisée des erreurs
export class ErrorInterceptor implements HttpInterceptor {
  intercept(req: HttpRequest<any>, next: HttpHandler) {
    return next.handle(req).pipe(
      catchError((error: HttpErrorResponse) => {
        switch (error.status) {
          case 400: this.notify.show('Requête invalide');         break;
          case 403: this.router.navigate(['/forbidden']);         break;
          case 500: this.notify.show('Erreur serveur interne');  break;
        }
        return throwError(() => error);
      })
    );
  }
}

// 3. Loading Interceptor — spinner global automatique
export class LoadingInterceptor implements HttpInterceptor {
  intercept(req: HttpRequest<any>, next: HttpHandler) {
    this.loading.show();
    return next.handle(req).pipe(
      finalize(() => this.loading.hide())
    );
  }
}

// Enregistrés dans CoreModule — ordre explicite :
providers: [
  { provide: HTTP_INTERCEPTORS, useClass: JwtInterceptor,     multi: true },
  { provide: HTTP_INTERCEPTORS, useClass: ErrorInterceptor,   multi: true },
  { provide: HTTP_INTERCEPTORS, useClass: LoadingInterceptor, multi: true },
]
```

---

## CoreModule — guard d'import unique

```typescript
// core/core.module.ts
@NgModule({
  providers: [ /* services singleton */ ]
})
export class CoreModule {
  // Empêche d'importer CoreModule dans un Feature Module
  // (ce qui créerait plusieurs instances des services)
  constructor(@Optional() @SkipSelf() parentModule: CoreModule) {
    if (parentModule) {
      throw new Error(
        'CoreModule est déjà chargé. Importez-le uniquement dans AppModule.'
      );
    }
  }
}
```

---

## Tests unitaires — services isolés

```typescript
// user-api.service.spec.ts
describe('UserApiService', () => {
  let service: UserApiService;
  let httpMock: HttpTestingController;

  beforeEach(() => {
    TestBed.configureTestingModule({
      imports: [HttpClientTestingModule],
      providers: [UserApiService]
    });
    service = TestBed.inject(UserApiService);
    httpMock = TestBed.inject(HttpTestingController);
  });

  afterEach(() => httpMock.verify());

  it('should fetch users from API', () => {
    const mockUsers: User[] = [{ id: 1, firstName: 'Abderrahmane', lastName: 'Elouafi' }];

    service.getUsers().subscribe(users => {
      expect(users.length).toBe(1);
      expect(users[0].firstName).toBe('Abderrahmane');
    });

    const req = httpMock.expectOne('/api/users');
    expect(req.request.method).toBe('GET');
    req.flush(mockUsers);
  });

  it('should handle 500 error gracefully', () => {
    service.getUsers().subscribe({
      error: err => expect(err.status).toBe(500)
    });
    httpMock.expectOne('/api/users').flush('Server Error', {
      status: 500, statusText: 'Internal Server Error'
    });
  });
});
```

---

## Ce que j'ai appris

Le principe SRP est celui qu'on viole le plus naturellement — pas par
incompétence, mais parce qu'au moment d'écrire le code, "ajouter juste
cette petite chose ici" semble raisonnable. Le problème apparaît
3 mois plus tard quand on veut écrire un test unitaire et qu'on réalise
que le service a 5 dépendances qu'il faut toutes mocker.

La règle pratique que j'applique : si un test unitaire nécessite plus de
3 mocks pour fonctionner, le service fait probablement trop de choses.
C'est le signal pour le découper.

---

*Projet réalisé dans le cadre de ma formation ingénieur — ENSET Mohammedia*
*Par **Abderrahmane Elouafi** · [LinkedIn](https://www.linkedin.com/in/abderrahmane-elouafi-43226736b/) · [Portfolio](https://my-first-porfolio-six.vercel.app/)*
