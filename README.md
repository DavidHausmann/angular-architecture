# Arquitetura Front-end Angular
Proposta de arquitetura para projetos front-end utilizando angular.

Versão: 1.0.0  
Última atualização: 30/10/2025  
Autores: David William Hausmann

## 📌 Convenções de Nomenclatura

| Elemento | Padrão de Nome | Exemplo |
|----------|----------------|---------|
| Componentes | nome.component.ts | statement-transaction-details.component.ts |
| Serviços | nome.service.ts | auth.service.ts |
| Models | nome.model.ts / nome.interface.ts | statement.model.ts |
| Páginas | nome-page.component.ts | statement-page.component.ts |
| Pipes | nome.pipe.ts | statement-transaction-status.pipe.ts |

## 🎯 Responsabilidades por Pasta

### core/
● Serviços únicos e globais  
● Interceptors  
● Guards  
● Configurações de providers  
● **Error Handlers e Strategies para tratamento de erros globais**

### shared/
● Pipes, directives e componentes reutilizáveis  
● Devem ser genéricos e independentes de regra de negócio

### layout/
● Componentes de estrutura visual da aplicação  
● Ex: MainLayoutComponent, HeaderComponent, SidebarComponent

### features/
● Cada subpasta representa uma feature do domínio  
● Isoladas, com seus próprios componentes, serviços, models, etc.

## 🔀 Roteamento
● Roteamento central em app-routing.module.ts  
● Cada feature deve ter seu routing.module.ts próprio  
● Usar Lazy Loading em cada feature

**Exemplo /app-routing.module.ts**
```typescript
{
  path: 'pagar',
  loadChildren: () =>
    import('./pages/pay/pay.module').then(m => m.PayModule),
  canActivate: [AuthGuard], // aplica o guard no carregamento da rota
  data: {
    roles: [AccountSignUpStatus.active, AccountPermissions.hasPayment],
  },
},
```

## 🎨 Padrões para Estilos (SCSS)
● Organização baseada em BEM (block__element--modifier)  
● Um SCSS por componente  
● SCSS globais ficam em src/styles/  
● Criar:  
  ○ _variables.scss  
  ○ _mixins.scss  
  ○ _themes.scss  
  ○ styles.scss

**Importar no angular.json:**
```json
"styles": ["src/styles/styles.scss"]
```

## ✍️ Padrões de Código
● Sempre utilizar TypeScript fortemente tipado  
● Evitar any, null, e valores mágicos  
● Usar async pipe sempre que possível  
● Usar RxJS com operadores como takeUntil, first, tap para controle  
● Sempre destruir subscriptions (takeUntil ou untilDestroyed)  
● Separar lógica de negócio em services

## ⚡ Signals (Angular 16+)
● Utilizar Signals para gerenciamento de estado reativo moderno  
● Signals oferecem melhor performance que observables em muitos casos  
● Usar `signal()` para estado local de componentes  
● Usar `computed()` para valores derivados que dependem de outros signals  
● Usar `effect()` para side effects baseados em mudanças de signals  
● Migrar gradualmente de RxJS para Signals onde apropriado  
● Combinar Signals com RxJS quando necessário (`toObservable()`, `toSignal()`)  

**Exemplo de uso de Signals:**
```typescript
@Component({
  selector: 'app-counter',
  template: `
    <div>
      <p>Contador: {{ count() }}</p>
      <p>Dobrado: {{ doubledCount() }}</p>
      <button (click)="increment()" [attr.data-test]="'increment-btn'">+</button>
      <button (click)="decrement()" [attr.data-test]="'decrement-btn'">-</button>
    </div>
  `
})
export class CounterComponent {
  // Signal básico
  count = signal(0);
  
  // Signal computado (derivado)
  doubledCount = computed(() => this.count() * 2);
  
  // Effect para side effects
  constructor() {
    effect(() => {
      console.log(`Contador mudou para: ${this.count()}`);
    });
  }
  
  increment() {
    this.count.update(value => value + 1);
  }
  
  decrement() {
    this.count.update(value => value - 1);
  }
}
```

**Quando usar Signals vs RxJS:**
● **Signals:** Estado local, valores derivados simples, performance crítica  
● **RxJS:** HTTP requests, operações assíncronas complexas, streams de eventos  

## � Tratamento de Erros Globais
● Implementar Error Handler global para capturar erros não tratados  
● Criar strategies para mapeamento de códigos de erro do backend  
● Usar interceptors para tratamento padronizado de erros HTTP  
● Implementar logging centralizado de erros  
● Criar dicionário de mensagens de erro user-friendly  
● Usar toast/notification service para exibir erros ao usuário  

**Estrutura recomendada em core/errors/:**
```
core/
└── errors/
    ├── global-error-handler.ts
    ├── error-interceptor.ts
    ├── error-mapping.service.ts
    ├── error-notification.service.ts
    └── models/
        └── error-response.model.ts
```

**Exemplo de Error Strategy:**
```typescript
// core/errors/error-mapping.service.ts
@Injectable({
  providedIn: 'root'
})
export class ErrorMappingService {
  private readonly errorMap = new Map<string, string>([
    ['ERR0006', 'Agendamento deve ser posterior ao dia atual'],
    ['ERR0001', 'Usuário não autorizado para esta operação'],
    ['ERR0002', 'Sessão expirada. Faça login novamente'],
    ['ERR0003', 'Dados inválidos fornecidos'],
    // ... mais mapeamentos
  ]);

  getUserFriendlyMessage(errorCode: string, fallbackMessage?: string): string {
    return this.errorMap.get(errorCode) || fallbackMessage || 'Erro inesperado. Tente novamente.';
  }

  logError(error: any, context?: string): void {
    console.error(`[${context || 'APP'}]`, error);
    // Integrar com serviço de logging (ex: Sentry, LogRocket)
  }
}

// core/errors/error-interceptor.ts
@Injectable()
export class ErrorInterceptor implements HttpInterceptor {
  constructor(
    private errorMapping: ErrorMappingService,
    private notification: ErrorNotificationService
  ) {}

  intercept(req: HttpRequest<any>, next: HttpHandler): Observable<HttpEvent<any>> {
    return next.handle(req).pipe(
      catchError((error: HttpErrorResponse) => {
        if (error.status >= 400) {
          const errorCode = error.error?.code;
          const message = this.errorMapping.getUserFriendlyMessage(
            errorCode, 
            error.error?.message
          );
          
          this.notification.showError(message);
          this.errorMapping.logError(error, 'HTTP_INTERCEPTOR');
        }
        
        return throwError(() => error);
      })
    );
  }
}
```

**Configuração no app.module.ts:**
```typescript
{
  provide: HTTP_INTERCEPTORS,
  useClass: ErrorInterceptor,
  multi: true
},
{
  provide: ErrorHandler,
  useClass: GlobalErrorHandler
}
```

## 📦 Boas Práticas
● Componentes pequenos e reutilizáveis (SRP)  
● Dividir entre containers (pages) e components  
● Isolar lógica em services  
● Reutilizar pipes e directives via shared  
● Evitar lógica de negócio em componentes  
● Aplicar testes unitários (pelo menos para serviços e pipes)

## 🔄 Reutilização de Código
● Criar componentes genéricos e reutilizáveis na pasta `shared/`  
● Componentes devem ser independentes de regras de negócio específicas  
● Usar @Input() e @Output() para tornar componentes configuráveis  
● Implementar interfaces consistentes para props similares  
● Utilizar Design System quando disponível para garantir consistência visual  
● Criar bibliotecas de componentes para elementos comuns (botões, inputs, modais)  
● Documentar componentes reutilizáveis com exemplos de uso  
● Aplicar versionamento semântico em bibliotecas de componentes  
● Usar Content Projection (ng-content) para componentes flexíveis  

**Exemplo de componente reutilizável:**
```typescript
@Component({
  selector: 'app-button',
  template: `
    <button 
      [class]="buttonClass" 
      [disabled]="disabled"
      (click)="onClick.emit($event)"
      [attr.data-test]="dataTest">
      <ng-content></ng-content>
    </button>
  `
})
export class ButtonComponent {
  @Input() variant: 'primary' | 'secondary' | 'danger' = 'primary';
  @Input() size: 'small' | 'medium' | 'large' = 'medium';
  @Input() disabled = false;
  @Input() dataTest?: string;
  @Output() onClick = new EventEmitter<Event>();

  get buttonClass(): string {
    return `btn btn--${this.variant} btn--${this.size}`;
  }
}
```

## 🧪 Testes
● *.spec.ts para cada componente/serviço/pure function  
● Usar Jasmine e Karma (padrão Angular)  
● Priorizar:  
  ○ Testes de serviços  
  ○ Pipes  
  ○ Componentes com lógica relevante

## 📚 Exemplo de Organização de uma Feature

```
features/
└── some-feature-dashboard/
    ├── pages/
    │   ├── some-feature-received-page/
    │   │   ├── some-feature-received-page.component.html
    │   │   ├── some-feature-received-page.component.scss
    │   │   └── some-feature-received-page.component.ts
    │   ├── some-feature-sales-page/
    │   │   └── (arquivos da page...)
    │   └── some-feature-to-receive-page/
    │       └── (arquivos da page...)
    ├── components/
    │   ├── some-feature-dashboard-actions/
    │   │   ├── some-feature-dashboard-actions.component.html
    │   │   ├── some-feature-dashboard-actions.component.scss
    │   │   └── some-feature-dashboard-actions.component.ts
    │   ├── some-feature-dashboard-title/
    │   │   └── (arquivos do componente...)
    │   └── some-feature-sales-tax/
    │       └── (arquivos do componente...)
    ├── services/
    │   ├── api/
    │   │   └── some-feature-dashboard.service.ts
    │   └── application/
    │       ├── some-feature-main-acquirers.service.ts
    │       ├── some-feature-received-dash.service.ts
    │       ├── some-feature-received-info.service.ts
    │       ├── some-feature-sales-by-modality.service.ts
    │       └── some-feature-sales-info.service.ts
    ├── models/
    │   └── some-feature-dashboard.model.ts
    ├── shared/
    │   └── some-feature-dashboard-styles.scss
    ├── some-feature-dashboard-routing.module.ts
    └── some-feature-dashboard.module.ts
```

## 📐 Padrão para tipagem de @Input() e @Output()

**Exemplo de padrão:**
● Sempre tipar explicitamente  
● Usar nomes verbosos e descritivos  
● Evitar any ou tipagens implícitas

```typescript
@Input() transaction: TransactionModel;
@Output() transactionClicked = new EventEmitter<number>();
```

## 🧼 Padrão de ordenação no arquivo TypeScript

**Sugestão de ordem:**
1. Imports (de Angular → libs → app) (@angular → 'rxjs' → 'src/app/features')
2. Decorators (@Component, etc)
3. Injeções no constructor
4. Properties públicas
5. Properties privadas
6. ngOnInit / lifecycle hooks
7. Métodos públicos
8. Métodos privados
9. Helpers

📌 **Motivo:** Ajuda a manter leitura consistente entre arquivos.

## 🛑 Fallback de rota (Página 404)
Tratar o redirecionamento de rota inexistente com a página home:

```typescript
{ path: '**', component: HomeComponent }
```

## ✍️ Criação de atributos de referência para ferramenta de automação de testes

Durante o desenvolvimento, ao criar blocos de conteúdo e principalmente elementos de ação (Botões, Campos de texto/formulários em geral), incluir um atributo para referenciar o elemento. Assim, a ferramenta de automação poderá identificar os pontos de ações e executar os testes. **Nome do atributo:** `data-test`.

```html
<input type="text" data-test="username-input" />
```

## 🔧 Ferramentas Recomendadas
● ESLint (@angular-eslint)  
● Prettier para formatação automática  
● Husky + lint-staged para garantir lint antes de commits  
● Compodoc para gerar documentação técnica automática

## 📑 Checklist para Criar uma Nova Feature
● Criar pasta em features/  
● Criar feature.module.ts e feature-routing.module.ts  
● Criar pasta pages/ e o componente principal  
● Criar components/ com subcomponentes necessários  
● Criar services/ e models/ se necessário  
● Registrar lazy loading em app-routing.module.ts  
● Adicionar testes unitários

## 📝 Considerações Finais
Este guia serve como base viva para desenvolvimento frontend em Angular no projeto. Qualquer sugestão ou melhoria deve ser discutida entre os devs e incorporada ao documento.

