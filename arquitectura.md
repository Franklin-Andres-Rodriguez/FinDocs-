# Especificaciones Avanzadas de UX/UI para FinDocs AI

## 1. Arquitectura de Componentes Frontend

### 1.1 Jerarquía de Componentes React

```
└── App
    ├── AppShell (Layout principal)
    │   ├── Header (Navigation, User Profile, Notifications)
    │   ├── Sidebar (Navigation secundaria contextual)
    │   └── ContentArea (Área principal dinámica)
    │
    ├── DashboardModule
    │   ├── MetricsPanel (KPIs de uso del sistema)
    │   ├── RecentDocumentsList (Documentos recientes + estados)
    │   └── ActivityTimeline (Actividad reciente + estado)
    │
    ├── DocumentModule
    │   ├── DocumentUploader (Drag & drop + integraciones)
    │   ├── ProcessingMonitor (Estado detallado por fases)
    │   │   ├── ProcessingStage (Componente individual por fase)
    │   │   └── ProcessingProgress (Barra + estimación de tiempo)
    │   ├── DocumentViewer (Vista PDF con highlighting)
    │   │   ├── PageNavigator
    │   │   ├── TableHighlighter
    │   │   └── SectionNavigator
    │   └── DocumentManager (CRUD de documentos)
    │
    ├── AnalysisModule
    │   ├── SplitView (Vista dividida documento/análisis)
    │   │   ├── OriginalDocumentPane
    │   │   └── AnalysisPane
    │   ├── FinancialMetricsPanel (Métricas extraídas)
    │   │   ├── MetricCard (Con indicador de confianza)
    │   │   └── ConfidenceIndicator (Sistema visual de confianza)
    │   ├── RatiosPanel (Ratios calculados según Damodaran)
    │   ├── VisualizationPanel (Gráficos interactivos)
    │   │   ├── TrendChart
    │   │   ├── ComparisonChart
    │   │   └── RatioVisualizer
    │   └── AnalysisExporter (Exportación personalizable)
    │
    ├── ConversationalModule
    │   ├── ChatInterface
    │   │   ├── MessageList (Historial de intercambios)
    │   │   ├── QueryInput (Input con autocompletado)
    │   │   └── SuggestedQueries (Consultas contextuales)
    │   ├── DocumentCitations (Referencias a fuentes)
    │   └── ContextPanel (Contexto activo de la conversación)
    │
    └── CommonComponents (Reutilizables en toda la aplicación)
        ├── LoadingStates (Spinners, Skeletons, Progress)
        ├── ErrorHandlers (Diferentes niveles de error)
        ├── EmptyStates (Estados vacíos contextuales)
        ├── Notifications (Sistema de alertas y notificaciones)
        └── Modals (Sistema unificado de modales)
```

### 1.2 Estructura del Estado Redux

```javascript
// Estructura del estado global
{
  auth: {
    user: { id, name, email, role, preferences },
    isAuthenticated: boolean,
    permissions: string[],
    status: 'idle' | 'loading' | 'succeeded' | 'failed',
    error: null | string
  },
  
  documents: {
    items: {
      [id]: {
        id: string,
        name: string,
        uploadDate: Date,
        fileSize: number,
        fileType: string,
        status: 'pending' | 'processing' | 'completed' | 'failed',
        metadata: { company, period, docType, ... },
        processingProgress: {
          currentStage: 'chunking' | 'extraction' | 'analysis',
          stageProgress: number, // 0-100
          overallProgress: number, // 0-100
          estimatedTimeRemaining: number, // segundos
          startedAt: Date,
          completedStages: string[]
        },
        error: null | { code: string, message: string, details: any }
      }
    },
    status: 'idle' | 'loading' | 'succeeded' | 'failed',
    currentDocumentId: string | null,
    filters: { status, docType, dateRange, ... },
    pagination: { page, limit, total },
    error: null | string
  },
  
  financialMetrics: {
    byDocumentId: {
      [documentId]: {
        metrics: {
          [metricId]: {
            id: string,
            name: string,
            value: number,
            unit: string,
            period: string,
            confidence: number, // 0-100
            source: { page, section, table },
            history: { period: string, value: number }[]
          }
        },
        ratios: {
          [ratioId]: {
            id: string,
            name: string,
            category: 'liquidity' | 'profitability' | 'solvency',
            value: number,
            formula: string,
            benchmark: number,
            trend: { period: string, value: number }[]
          }
        },
        tables: {
          [tableId]: {
            id: string,
            title: string,
            location: { page, position },
            structure: {}, // Estructura normalizada de la tabla
            confidence: number // 0-100
          }
        },
        status: 'idle' | 'loading' | 'succeeded' | 'failed',
        lastUpdated: Date,
        error: null | string
      }
    }
  },
  
  conversation: {
    byDocumentId: {
      [documentId]: {
        messages: [
          {
            id: string,
            sender: 'user' | 'assistant',
            text: string,
            timestamp: Date,
            references: { type, source, location }[]
          }
        ],
        context: {
          activeSections: string[],
          activeMetrics: string[],
          focusedPeriod: string
        },
        status: 'idle' | 'loading' | 'succeeded' | 'failed',
        error: null | string
      }
    },
    currentConversationId: string | null
  },
  
  ui: {
    theme: 'light' | 'dark' | 'auto',
    layout: {
      sidebarExpanded: boolean,
      currentView: string,
      splitViewRatio: number // 0-100
    },
    modals: {
      activeModal: string | null,
      modalData: any
    },
    notifications: [
      { id, type, message, timestamp, read, actions }
    ]
  }
}
```

### 1.3 Acciones Redux Principales

```javascript
// Acciones para procesamiento de documentos asíncrono
const documentActions = {
  // Carga de documento
  uploadDocumentRequest: (file, metadata) => ({ type: 'documents/uploadRequest', payload: { file, metadata } }),
  uploadDocumentSuccess: (document) => ({ type: 'documents/uploadSuccess', payload: document }),
  uploadDocumentFailure: (error) => ({ type: 'documents/uploadFailure', payload: error }),
  
  // Monitoreo de procesamiento
  fetchProcessingStatus: (documentId) => ({ type: 'documents/fetchProcessingStatus', payload: documentId }),
  updateProcessingProgress: (documentId, progress) => ({ type: 'documents/updateProgress', payload: { documentId, progress } }),
  
  // Manejo de documentos
  fetchDocuments: (filters, pagination) => ({ type: 'documents/fetchRequest', payload: { filters, pagination } }),
  selectDocument: (documentId) => ({ type: 'documents/selectDocument', payload: documentId }),
  deleteDocument: (documentId) => ({ type: 'documents/deleteRequest', payload: documentId })
};

// Acciones para métricas financieras
const metricsActions = {
  fetchMetrics: (documentId) => ({ type: 'financialMetrics/fetchRequest', payload: documentId }),
  updateMetricConfidence: (documentId, metricId, feedback) => ({ 
    type: 'financialMetrics/updateConfidence', 
    payload: { documentId, metricId, feedback } 
  }),
  fetchComparativeMetrics: (documentIds, metricIds) => ({ 
    type: 'financialMetrics/fetchComparative', 
    payload: { documentIds, metricIds } 
  })
};

// Acciones para conversaciones
const conversationActions = {
  sendMessage: (documentId, text) => ({ type: 'conversation/sendMessage', payload: { documentId, text } }),
  receiveMessage: (documentId, message) => ({ type: 'conversation/receiveMessage', payload: { documentId, message } }),
  fetchConversationHistory: (documentId) => ({ type: 'conversation/fetchHistory', payload: documentId }),
  updateContext: (documentId, context) => ({ type: 'conversation/updateContext', payload: { documentId, context } })
};
```

### 1.4 Selectores Redux Principales

```javascript
// Selectores para documentos
const documentSelectors = {
  selectAllDocuments: (state) => Object.values(state.documents.items),
  selectDocumentById: (state, documentId) => state.documents.items[documentId],
  selectCurrentDocument: (state) => {
    const { currentDocumentId } = state.documents;
    return currentDocumentId ? state.documents.items[currentDocumentId] : null;
  },
  selectDocumentProcessingStatus: (state, documentId) => {
    const document = state.documents.items[documentId];
    return document ? document.processingProgress : null;
  },
  selectFilteredDocuments: (state) => {
    const { items, filters } = state.documents;
    return Object.values(items).filter(doc => {
      // Aplicar filtros
      if (filters.status && doc.status !== filters.status) return false;
      if (filters.docType && doc.metadata.docType !== filters.docType) return false;
      // Más filtros...
      return true;
    });
  }
};

// Selectores para métricas financieras
const metricSelectors = {
  selectMetricsByDocument: (state, documentId) => {
    const documentMetrics = state.financialMetrics.byDocumentId[documentId];
    return documentMetrics ? documentMetrics.metrics : {};
  },
  selectRatiosByDocument: (state, documentId) => {
    const documentMetrics = state.financialMetrics.byDocumentId[documentId];
    return documentMetrics ? documentMetrics.ratios : {};
  },
  selectMetricById: (state, documentId, metricId) => {
    const documentMetrics = state.financialMetrics.byDocumentId[documentId];
    return documentMetrics && documentMetrics.metrics[metricId];
  },
  selectMetricsByConfidence: (state, documentId, threshold = 80) => {
    const metrics = metricSelectors.selectMetricsByDocument(state, documentId);
    return Object.values(metrics).filter(metric => metric.confidence >= threshold);
  },
  selectLowConfidenceMetrics: (state, documentId, threshold = 80) => {
    const metrics = metricSelectors.selectMetricsByDocument(state, documentId);
    return Object.values(metrics).filter(metric => metric.confidence < threshold);
  }
};

// Selectores para conversaciones
const conversationSelectors = {
  selectConversationByDocument: (state, documentId) => {
    return state.conversation.byDocumentId[documentId] || { messages: [], context: {} };
  },
  selectActiveContext: (state, documentId) => {
    const conversation = state.conversation.byDocumentId[documentId];
    return conversation ? conversation.context : {};
  },
  selectLastMessage: (state, documentId) => {
    const conversation = state.conversation.byDocumentId[documentId];
    if (!conversation || conversation.messages.length === 0) return null;
    return conversation.messages[conversation.messages.length - 1];
  }
};
```

## 2. Especificaciones de Integración API

### 2.1 Integración con Endpoints API

```javascript
// Cliente API para integración con el backend
const apiClient = {
  // Endpoints de documentos
  documents: {
    upload: async (file, metadata) => {
      const formData = new FormData();
      formData.append('file', file);
      formData.append('metadata', JSON.stringify(metadata));
      
      return fetch('/api/documents', {
        method: 'POST',
        body: formData,
        headers: {
          'Authorization': `Bearer ${getAuthToken()}`
        }
      }).then(handleResponse);
    },
    
    getAll: async (filters = {}, pagination = { page: 1, limit: 20 }) => {
      const queryParams = new URLSearchParams();
      Object.entries(filters).forEach(([key, value]) => {
        if (value) queryParams.append(key, value);
      });
      queryParams.append('page', pagination.page);
      queryParams.append('limit', pagination.limit);
      
      return fetch(`/api/documents?${queryParams.toString()}`, {
        headers: {
          'Authorization': `Bearer ${getAuthToken()}`
        }
      }).then(handleResponse);
    },
    
    getById: async (documentId) => {
      return fetch(`/api/documents/${documentId}`, {
        headers: {
          'Authorization': `Bearer ${getAuthToken()}`
        }
      }).then(handleResponse);
    },
    
    delete: async (documentId) => {
      return fetch(`/api/documents/${documentId}`, {
        method: 'DELETE',
        headers: {
          'Authorization': `Bearer ${getAuthToken()}`
        }
      }).then(handleResponse);
    },
    
    getProcessingStatus: async (documentId) => {
      return fetch(`/api/documents/${documentId}/status`, {
        headers: {
          'Authorization': `Bearer ${getAuthToken()}`
        }
      }).then(handleResponse);
    },
    
    getChunks: async (documentId, section = null, pagination = { page: 1, limit: 20 }) => {
      const queryParams = new URLSearchParams();
      if (section) queryParams.append('section', section);
      queryParams.append('page', pagination.page);
      queryParams.append('limit', pagination.limit);
      
      return fetch(`/api/documents/${documentId}/chunks?${queryParams.toString()}`, {
        headers: {
          'Authorization': `Bearer ${getAuthToken()}`
        }
      }).then(handleResponse);
    }
  },
  
  // Endpoints de métricas financieras
  financialMetrics: {
    getMetrics: async (documentId, type = 'all', period = null) => {
      const queryParams = new URLSearchParams();
      queryParams.append('type', type);
      if (period) queryParams.append('period', period);
      
      return fetch(`/api/documents/${documentId}/metrics?${queryParams.toString()}`, {
        headers: {
          'Authorization': `Bearer ${getAuthToken()}`
        }
      }).then(handleResponse);
    },
    
    getTables: async (documentId, type = null, page = null) => {
      const queryParams = new URLSearchParams();
      if (type) queryParams.append('type', type);
      if (page) queryParams.append('page', page);
      
      return fetch(`/api/documents/${documentId}/tables?${queryParams.toString()}`, {
        headers: {
          'Authorization': `Bearer ${getAuthToken()}`
        }
      }).then(handleResponse);
    },
    
    getRatios: async (documentId, category = 'all') => {
      const queryParams = new URLSearchParams();
      queryParams.append('category', category);
      
      return fetch(`/api/documents/${documentId}/ratios?${queryParams.toString()}`, {
        headers: {
          'Authorization': `Bearer ${getAuthToken()}`
        }
      }).then(handleResponse);
    },
    
    getTrends: async (documentId, metric, period = 'annual', duration = null) => {
      const queryParams = new URLSearchParams();
      queryParams.append('metric', metric);
      queryParams.append('period', period);
      if (duration) queryParams.append('duration', duration);
      
      return fetch(`/api/documents/${documentId}/trends?${queryParams.toString()}`, {
        headers: {
          'Authorization': `Bearer ${getAuthToken()}`
        }
      }).then(handleResponse);
    },
    
    provideMetricFeedback: async (documentId, metricId, feedback) => {
      return fetch(`/api/documents/${documentId}/metrics/${metricId}/feedback`, {
        method: 'POST',
        headers: {
          'Content-Type': 'application/json',
          'Authorization': `Bearer ${getAuthToken()}`
        },
        body: JSON.stringify(feedback)
      }).then(handleResponse);
    }
  },
  
  // Endpoints de IA conversacional
  conversation: {
    query: async (documentId, query, context = {}, history = []) => {
      return fetch(`/api/documents/${documentId}/query`, {
        method: 'POST',
        headers: {
          'Content-Type': 'application/json',
          'Authorization': `Bearer ${getAuthToken()}`
        },
        body: JSON.stringify({ query, context, history })
      }).then(handleResponse);
    },
    
    comparativeQuery: async (query, documentIds, context = {}) => {
      return fetch(`/api/analysis/query`, {
        method: 'POST',
        headers: {
          'Content-Type': 'application/json',
          'Authorization': `Bearer ${getAuthToken()}`
        },
        body: JSON.stringify({ query, documentIds, context })
      }).then(handleResponse);
    },
    
    getConversationHistory: async (conversationId) => {
      return fetch(`/api/conversations/${conversationId}`, {
        headers: {
          'Authorization': `Bearer ${getAuthToken()}`
        }
      }).then(handleResponse);
    },
    
    saveAnnotation: async (documentId, annotation) => {
      return fetch(`/api/documents/${documentId}/annotations`, {
        method: 'POST',
        headers: {
          'Content-Type': 'application/json',
          'Authorization': `Bearer ${getAuthToken()}`
        },
        body: JSON.stringify(annotation)
      }).then(handleResponse);
    },
    
    exportAnalysis: async (documentId, format = 'pdf', sections = []) => {
      const queryParams = new URLSearchParams();
      queryParams.append('format', format);
      sections.forEach(section => queryParams.append('sections', section));
      
      return fetch(`/api/documents/${documentId}/export?${queryParams.toString()}`, {
        headers: {
          'Authorization': `Bearer ${getAuthToken()}`
        }
      }).then(response => {
        if (!response.ok) {
          return response.json().then(err => { throw err; });
        }
        return response.blob();
      });
    }
  }
};

// Utilidad para manejar respuestas
const handleResponse = async (response) => {
  if (!response.ok) {
    const errorData = await response.json();
    throw {
      status: response.status,
      message: errorData.message || 'Error en la solicitud',
      details: errorData.details || {}
    };
  }
  return response.json();
};

// Utilidad para obtener token de autenticación
const getAuthToken = () => {
  return localStorage.getItem('authToken');
};
```

### 2.2 Estrategia de Manejo de Errores

```javascript
// Middleware Redux para manejo centralizado de errores de API
const apiErrorMiddleware = store => next => action => {
  const result = next(action);
  
  // Detectar acciones de error de API
  if (action.type.endsWith('/failure') && action.payload) {
    const { status, message, details } = action.payload;
    
    // Estrategia basada en código de error
    switch (status) {
      case 401: // No autorizado
        store.dispatch({
          type: 'auth/logout',
          payload: { reason: 'session_expired' }
        });
        store.dispatch({
          type: 'ui/showNotification',
          payload: {
            type: 'error',
            message: 'Tu sesión ha expirado. Por favor, inicia sesión nuevamente.'
          }
        });
        break;
        
      case 403: // Prohibido (permisos)
        store.dispatch({
          type: 'ui/showNotification',
          payload: {
            type: 'error',
            message: 'No tienes permisos para realizar esta acción.'
          }
        });
        break;
        
      case 404: // No encontrado
        store.dispatch({
          type: 'ui/showNotification',
          payload: {
            type: 'error',
            message: 'El recurso solicitado no existe o ha sido eliminado.'
          }
        });
        break;
        
      case 413: // Entidad demasiado grande
        store.dispatch({
          type: 'ui/showNotification',
          payload: {
            type: 'error',
            message: 'El archivo es demasiado grande. El tamaño máximo permitido es 50MB.'
          }
        });
        break;
        
      case 429: // Demasiadas solicitudes
        store.dispatch({
          type: 'ui/showNotification',
          payload: {
            type: 'warning',
            message: 'Has realizado demasiadas solicitudes. Por favor, espera unos minutos antes de intentarlo nuevamente.'
          }
        });
        break;
        
      case 500: // Error del servidor
        store.dispatch({
          type: 'ui/showNotification',
          payload: {
            type: 'error',
            message: 'Ha ocurrido un error en el servidor. Nuestro equipo ha sido notificado.',
            actions: [
              {
                label: 'Reintentar',
                action: action.meta?.originalAction // Reintentar la acción original
              }
            ]
          }
        });
        break;
        
      default:
        // Errores específicos de la aplicación (códigos personalizados)
        if (details.code === 'document_processing_failed') {
          store.dispatch({
            type: 'ui/showNotification',
            payload: {
              type: 'error',
              message: 'Error en el procesamiento del documento. Verifica el formato y vuelve a intentar.',
              actions: [
                {
                  label: 'Ver detalles',
                  action: {
                    type: 'ui/openModal',
                    payload: {
                      modalType: 'errorDetails',
                      modalProps: { error: details }
                    }
                  }
                }
              ]
            }
          });
        } else {
          // Error genérico
          store.dispatch({
            type: 'ui/showNotification',
            payload: {
              type: 'error',
              message: message || 'Ha ocurrido un error inesperado.'
            }
          });
        }
    }
  }
  
  return result;
};

// Componente para manejo de errores a nivel de UI
const ErrorBoundary = ({ children, fallbackUI, onError }) => {
  const [error, setError] = useState(null);
  
  useEffect(() => {
    if (error && onError) {
      onError(error);
    }
  }, [error, onError]);
  
  if (error) {
    return fallbackUI ? (
      <ErrorFallback 
        error={error} 
        onReset={() => setError(null)} 
      />
    ) : (
      <div className="error-container">
        <h3>Algo salió mal</h3>
        <p>{error.message || 'Error inesperado'}</p>
        <button onClick={() => setError(null)}>
          Reintentar
        </button>
      </div>
    );
  }
  
  return (
    <ErrorBoundaryComponent onError={setError}>
      {children}
    </ErrorBoundaryComponent>
  );
};

// Estados de error en componentes
const ComponentErrorStates = {
  // Para cargas iniciales fallidas
  LoadingError: ({ error, onRetry }) => (
    <div className="error-state">
      <Icon name="alert-circle" size={48} className="text-error mb-4" />
      <h4>Error al cargar datos</h4>
      <p className="text-gray-500 mb-4">{error.message}</p>
      <Button onClick={onRetry}>Reintentar</Button>
    </div>
  ),
  
  // Para acciones fallidas
  ActionError: ({ error, onClose }) => (
    <div className="notification-error">
      <div className="flex items-center">
        <Icon name="alert-triangle" className="text-error mr-2" />
        <span>{error.message}</span>
      </div>
      <button className="ml-auto" onClick={onClose}>
        <Icon name="x" size={16} />
      </button>
    </div>
  ),
  
  // Para formularios con validación
  FormError: ({ errors }) => (
    <div className="form-errors">
      <ul>
        {Object.entries(errors).map(([field, message]) => (
          <li key={field} className="text-error text-sm flex items-center">
            <Icon name="alert-circle" size={12} className="mr-1" />
            {message}
          </li>
        ))}
      </ul>
    </div>
  )
};
```

### 2.3 Estrategia de Cache y Rendimiento

```javascript
// Configuración de cache para RTK Query
const apiSlice = createApi({
  reducerPath: 'api',
  baseQuery: fetchBaseQuery({ 
    baseUrl: '/api/',
    prepareHeaders: (headers, { getState }) => {
      const token = getAuthToken();
      if (token) {
        headers.set('authorization', `Bearer ${token}`);
      }
      return headers;
    }
  }),
  tagTypes: ['Document', 'Metrics', 'Ratios', 'Conversation'],
  endpoints: (builder) => ({
    // Endpoints con políticas de cache específicas
    getDocuments: builder.query({
      query: (args) => {
        const { filters = {}, pagination = {} } = args || {};
        // Construir query params
        return {
          url: 'documents',
          params: { ...filters, ...pagination }
        };
      },
      // Invalidación de cache para documento específico
      providesTags: (result) => 
        result
          ? [
              ...result.items.map(({ id }) => ({ type: 'Document', id })),
              { type: 'Document', id: 'LIST' }
            ]
          : [{ type: 'Document', id: 'LIST' }]
    }),
    
    getDocumentById: builder.query({
      query: (id) => `documents/${id}`,
      providesTags: (result, error, id) => [{ type: 'Document', id }]
    }),
    
    // Cache para métricas financieras con tiempo de vida
    getFinancialMetrics: builder.query({
      query: ({ documentId, type, period }) => ({
        url: `documents/${documentId}/metrics`,
        params: { type, period }
      }),
      providesTags: (result, error, { documentId }) => [
        { type: 'Metrics', id: documentId }
      ],
      // TTL de 5 minutos para métricas
      keepUnusedDataFor: 300
    }),
    
    // Cache agresiva para datos que no cambian frecuentemente
    getDocumentRatios: builder.query({
      query: ({ documentId, category }) => ({
        url: `documents/${documentId}/ratios`,
        params: { category }
      }),
      providesTags: (result, error, { documentId }) => [
        { type: 'Ratios', id: documentId }
      ],
      // TTL de 1 hora para ratios
      keepUnusedDataFor: 3600
    })
  })
});

// Hook personalizado para gestionar estados de carga y error
const useApiQuery = (queryHook, ...args) => {
  const result = queryHook(...args);
  
  // Mapeo de estados común
  const mappedResult = {
    ...result,
    isInitialLoading: result.isLoading && !result.isFetching,
    mappedError: result.error ? mapApiError(result.error) : null,
    // Refetch con backoff exponencial para reintentos
    retryWithBackoff: async (maxRetries = 3) => {
      let retries = 0;
      let lastError;
      
      while (retries < maxRetries) {
        try {
          await result.refetch();
          return true;
        } catch (error) {
          lastError = error;
          const backoffTime = Math.pow(2, retries) * 1000; // 1s, 2s, 4s...
          await new Promise(resolve => setTimeout(resolve, backoffTime));
          retries++;
        }
      }
      
      throw lastError;
    }
  };
  
  return mappedResult;
};

// Memoización para componentes pesados
const MemoizedFinancialTable = React.memo(({ data, onCellClick }) => {
  // Implementación del componente
}, (prevProps, nextProps) => {
  // Comparación personalizada para evitar re-renderizados innecesarios
  return (
    prevProps.data.length === nextProps.data.length &&
    prevProps.data.every((item, index) => 
      item.id === nextProps.data[index].id && 
      item.value === nextProps.data[index].value
    )
  );
});
```

## 3. Implementación de Visualizaciones Financieras

### 3.1 Especificaciones Técnicas de Visualizaciones

```javascript
// Configuración base para todas las visualizaciones
const visualizationConfig = {
  // Paleta de colores para categorías
  colorPalette: {
    primary: ['#0055B8', '#1976D2', '#4D9FE0', '#7FC8F8'],
    secondary: ['#00A878', '#10B981', '#4ADE80', '#86EFAC'],
    accent: ['#9333EA', '#A855F7', '#C084FC', '#DDB6FC'],
    status: {
      positive: '#00A878',
      neutral: '#7F8C96',
      negative: '#E63946',
      warning: '#F59E0B'
    }
  },
  
  // Configuración de tipografía
  typography: {
    title: {
      fontFamily: 'Inter, sans-serif',
      fontSize: '16px',
      fontWeight: 600,
      lineHeight: 1.4
    },
    subtitle: {
      fontFamily: 'Inter, sans-serif',
      fontSize: '14px',
      fontWeight: 500,
      lineHeight: 1.4
    },
    axis: {
      fontFamily: 'Inter, sans-serif',
      fontSize: '12px',
      fontWeight: 400,
      lineHeight: 1.2
    },
    tooltip: {
      fontFamily: 'Inter, sans-serif',
      fontSize: '12px',
      fontWeight: 400,
      lineHeight: 1.2
    },
    dataLabel: {
      fontFamily: 'Roboto Mono, monospace',
      fontSize: '12px',
      fontWeight: 500,
      lineHeight: 1
    }
  },
  
  // Tamaños y espaciados
  dimensions: {
    margin: { top: 24, right: 24, bottom: 40, left: 56 },
    height: 320,
    aspectRatio: 16/9,
    barWidth: 24,
    lineStrokeWidth: 2,
    pointRadius: 4
  },
  
  // Animaciones
  animations: {
    duration: 500,
    easing: 'cubic-bezier(0.4, 0, 0.2, 1)', // Material easing
    delay: 50
  },
  
  // Configuración de accesibilidad
  accessibility: {
    minContrastRatio: 4.5, // WCAG AA
    patterns: ['stripe', 'dots', 'grid'], // Para daltonismo
    ariaLabels: {
      chart: 'Gráfico financiero mostrando {title}',
      bar: 'Barra representando {value} para {category}',
      line: 'Tendencia de {metric} a lo largo del tiempo',
      point: 'Valor de {value} para {period}'
    }
  },
  
  // Configuración de interactividad
  interactivity: {
    tooltipDelay: 200,
    hoverHighlightOpacity: 0.8,
    selectedHighlightOpacity: 1,
    inactiveOpacity: 0.3,
    tapRadius: 24 // Para móviles
  }
};

// Componente de gráfico de tendencias (implementación con D3.js)
const TrendChart = ({
  data,
  metric,
  period = 'annual',
  benchmark,
  height = visualizationConfig.dimensions.height,
  showDataLabels = true,
  onPointClick,
  className
}) => {
  const svgRef = useRef(null);
  const tooltipRef = useRef(null);
  const containerRef = useRef(null);
  
  useEffect(() => {
    if (!data || data.length === 0) return;
    
    // Cálculo de dimensiones
    const containerWidth = containerRef.current.clientWidth;
    const width = containerWidth - visualizationConfig.dimensions.margin.left - visualizationConfig.dimensions.margin.right;
    const chartHeight = height - visualizationConfig.dimensions.margin.top - visualizationConfig.dimensions.margin.bottom;
    
    // Crear escalas
    const xScale = d3.scaleTime()
      .domain(d3.extent(data, d => new Date(d.period)))
      .range([0, width]);
    
    const yScale = d3.scaleLinear()
      .domain([
        d3.min(data, d => d.value) * 0.9, // 10% de padding inferior
        d3.max(data, d => d.value) * 1.1  // 10% de padding superior
      ])
      .range([chartHeight, 0]);
    
    // Crear generador de línea
    const line = d3.line()
      .x(d => xScale(new Date(d.period)))
      .y(d => yScale(d.value))
      .curve(d3.curveMonotoneX); // Curva suave
    
    // Limpiar SVG
    const svg = d3.select(svgRef.current);
    svg.selectAll('*').remove();
    
    // Crear grupo principal con márgenes
    const g = svg.append('g')
      .attr('transform', `translate(${visualizationConfig.dimensions.margin.left},${visualizationConfig.dimensions.margin.top})`);
    
    // Dibujar eje X
    const xAxis = g.append('g')
      .attr('transform', `translate(0,${chartHeight})`)
      .call(d3.axisBottom(xScale)
        .ticks(period === 'quarterly' ? d3.timeQuarter : d3.timeYear)
        .tickFormat(period === 'quarterly' ? 
          d => `Q${Math.floor(d.getMonth() / 3) + 1} ${d.getFullYear()}` : 
          d => d.getFullYear()
        )
      );
    
    // Estilizar eje X
    xAxis.selectAll('text')
      .style('font-family', visualizationConfig.typography.axis.fontFamily)
      .style('font-size', visualizationConfig.typography.axis.fontSize)
      .attr('dy', '0.7em');
    
    // Dibujar eje Y
    const yAxis = g.append('g')
      .call(d3.axisLeft(yScale)
        .ticks(5)
        .tickFormat(d => {
          // Formatear valores según tipo
          if (metric.unit === '%') return `${d}%`;
          if (metric.unit === '$') return formatCurrency(d);
          return d;
        })
      );
    
    // Estilizar eje Y
    yAxis.selectAll('text')
      .style('font-family', visualizationConfig.typography.axis.fontFamily)
      .style('font-size', visualizationConfig.typography.axis.fontSize);
    
    // Dibujar líneas de cuadrícula
    g.append('g')
      .attr('class', 'grid')
      .selectAll('line')
      .data(yScale.ticks(5))
      .enter()
      .append('line')
      .attr('x1', 0)
      .attr('x2', width)
      .attr('y1', d => yScale(d))
      .attr('y2', d => yScale(d))
      .attr('stroke', '#E9ECEF')
      .attr('stroke-dasharray', '3,3');
    
    // Dibujar línea de benchmark si existe
    if (benchmark) {
      g.append('line')
        .attr('x1', 0)
        .attr('x2', width)
        .attr('y1', yScale(benchmark.value))
        .attr('y2', yScale(benchmark.value))
        .attr('stroke', visualizationConfig.colorPalette.status.neutral)
        .attr('stroke-width', 1.5)
        .attr('stroke-dasharray', '5,5');
      
      // Etiqueta de benchmark
      g.append('text')
        .attr('x', width)
        .attr('y', yScale(benchmark.value) - 5)
        .attr('text-anchor', 'end')
        .style('font-family', visualizationConfig.typography.subtitle.fontFamily)
        .style('font-size', '11px')
        .style('fill', visualizationConfig.colorPalette.status.neutral)
        .text(benchmark.label);
    }
    
    // Dibujar línea principal
    g.append('path')
      .datum(data)
      .attr('fill', 'none')
      .attr('stroke', visualizationConfig.colorPalette.primary[0])
      .attr('stroke-width', visualizationConfig.dimensions.lineStrokeWidth)
      .attr('d', line)
      .style('opacity', 0)
      .transition()
      .duration(visualizationConfig.animations.duration)
      .style('opacity', 1);
    
    // Dibujar puntos
    const points = g.selectAll('.data-point')
      .data(data)
      .enter()
      .append('circle')
      .attr('class', 'data-point')
      .attr('cx', d => xScale(new Date(d.period)))
      .attr('cy', d => yScale(d.value))
      .attr('r', visualizationConfig.dimensions.pointRadius)
      .attr('fill', visualizationConfig.colorPalette.primary[0])
      .style('opacity', 0)
      .transition()
      .duration(visualizationConfig.animations.duration)
      .delay((d, i) => i * visualizationConfig.animations.delay)
      .style('opacity', 1);
    
    // Etiquetas de datos
    if (showDataLabels) {
      g.selectAll('.data-label')
        .data(data)
        .enter()
        .append('text')
        .attr('class', 'data-label')
        .attr('x', d => xScale(new Date(d.period)))
        .attr('y', d => yScale(d.value) - 10)
        .attr('text-anchor', 'middle')
        .style('font-family', visualizationConfig.typography.dataLabel.fontFamily)
        .style('font-size', visualizationConfig.typography.dataLabel.fontSize)
        .style('fill', visualizationConfig.colorPalette.primary[0])
        .style('opacity', 0)
        .text(d => {
          if (metric.unit === '%') return `${d.value.toFixed(1)}%`;
          if (metric.unit === '$') return formatCompactCurrency(d.value);
          return d.value.toLocaleString();
        })
        .transition()
        .duration(visualizationConfig.animations.duration)
        .delay((d, i) => i * visualizationConfig.animations.delay + 200)
        .style('opacity', 1);
    }
    
    // Título del gráfico
    svg.append('text')
      .attr('x', visualizationConfig.dimensions.margin.left)
      .attr('y', 16)
      .style('font-family', visualizationConfig.typography.title.fontFamily)
      .style('font-size', visualizationConfig.typography.title.fontSize)
      .style('font-weight', visualizationConfig.typography.title.fontWeight)
      .text(metric.name);
    
    // Configurar tooltip
    const tooltip = d3.select(tooltipRef.current);
    
    // Áreas interactivas (más grandes que los puntos para facilitar interacción)
    g.selectAll('.interaction-area')
      .data(data)
      .enter()
      .append('circle')
      .attr('class', 'interaction-area')
      .attr('cx', d => xScale(new Date(d.period)))
      .attr('cy', d => yScale(d.value))
      .attr('r', visualizationConfig.interactivity.tapRadius)
      .attr('fill', 'transparent')
      .style('cursor', 'pointer')
      .on('mouseover', (event, d) => {
        // Mostrar tooltip
        tooltip
          .style('display', 'block')
          .style('opacity', 0)
          .html(`
            <div class="font-medium">${formatPeriod(d.period, period)}</div>
            <div>${metric.name}: ${formatValue(d.value, metric.unit)}</div>
            ${d.change ? `<div class="${d.change > 0 ? 'text-success' : 'text-error'}">
              ${d.change > 0 ? '▲' : '▼'} ${Math.abs(d.change).toFixed(1)}%
            </div>` : ''}
          `)
          .style('left', `${event.pageX}px`)
          .style('top', `${event.pageY - 28}px`)
          .transition()
          .duration(200)
          .style('opacity', 1);
        
        // Resaltar punto
        d3.select(event.currentTarget.parentNode)
          .select(`.data-point:nth-child(${data.indexOf(d) + 1})`)
          .transition()
          .duration(100)
          .attr('r', visualizationConfig.dimensions.pointRadius * 1.5);
      })
      .on('mouseout', (event, d) => {
        // Ocultar tooltip
        tooltip
          .transition()
          .duration(200)
          .style('opacity', 0)
          .on('end', () => tooltip.style('display', 'none'));
        
        // Restaurar punto
        d3.select(event.currentTarget.parentNode)
          .select(`.data-point:nth-child(${data.indexOf(d) + 1})`)
          .transition()
          .duration(100)
          .attr('r', visualizationConfig.dimensions.pointRadius);
      })
      .on('click', (event, d) => {
        // Manejar clic en punto
        if (onPointClick) {
          onPointClick(d, data.indexOf(d));
        }
      });
    
  }, [data, metric, period, benchmark, height, showDataLabels, onPointClick]);
  
  // Renderizado del componente
  return (
    <div ref={containerRef} className={`trend-chart-container ${className || ''}`}>
      <svg 
        ref={svgRef} 
        width="100%" 
        height={height}
        aria-label={visualizationConfig.accessibility.ariaLabels.chart.replace('{title}', metric?.name || 'tendencia financiera')}
        role="img"
      />
      <div 
        ref={tooltipRef} 
        className="tooltip" 
        style={{ 
          position: 'absolute', 
          display: 'none',
          backgroundColor: 'white',
          border: '1px solid #DEE2E6',
          borderRadius: '4px',
          padding: '8px',
          pointerEvents: 'none',
          boxShadow: '0 2px 4px rgba(0,0,0,0.1)',
          zIndex: 1000
        }}
      />
    </div>
  );
};

// Componente de comparación financiera (implementación con D3.js)
const ComparisonChart = ({
  data,
  metric,
  comparison = 'yoy', // 'yoy', 'qoq', 'benchmark'
  height = visualizationConfig.dimensions.height,
  showLabels = true,
  onBarClick,
  className
}) => {
  // Implementación similar a TrendChart con barras comparativas
  // ...código de implementación aquí...
};

// Componente de visualización de ratios (implementación con D3.js)
const RatioVisualizer = ({
  data,
  benchmarks,
  category,
  height = visualizationConfig.dimensions.height,
  onRatioClick,
  className
}) => {
  // Implementación de visualización de ratios con benchmark
  // ...código de implementación aquí...
};
```

### 3.2 Consideraciones de Rendimiento para Visualizaciones

```javascript
// Optimizaciones para visualizaciones con grandes conjuntos de datos
const OptimizedFinancialVisualization = ({
  data,
  metric,
  period,
  ...props
}) => {
  // 1. Muestreo adaptativo de datos para conjuntos muy grandes
  const sampledData = useMemo(() => {
    if (data.length <= 50) return data; // Sin muestreo para conjuntos pequeños
    
    // Implementar algoritmo de muestreo que preserve extremos y tendencias
    return downsampleTimeseries(data, 50);
  }, [data]);
  
  // 2. Renderizado condicional basado en visibilidad
  const [isVisible, setIsVisible] = useState(false);
  const containerRef = useRef(null);
  
  useEffect(() => {
    const observer = new IntersectionObserver(
      ([entry]) => {
        setIsVisible(entry.isIntersecting);
      },
      { threshold: 0.1 }
    );
    
    if (containerRef.current) {
      observer.observe(containerRef.current);
    }
    
    return () => {
      if (containerRef.current) {
        observer.unobserve(containerRef.current);
      }
    };
  }, []);
  
  // 3. Virtualizacion para múltiples gráficos
  const renderChart = () => {
    if (!isVisible) {
      return <div style={{ height }} />;
    }
    
    return (
      <TrendChart
        data={sampledData}
        metric={metric}
        period={period}
        {...props}
      />
    );
  };
  
  // 4. Web Workers para cálculos pesados
  const [processedData, setProcessedData] = useState(null);
  
  useEffect(() => {
    if (!isVisible || !data) return;
    
    // Crear Web Worker para cálculos pesados
    const worker = new Worker('/workers/financial-calculations.js');
    
    worker.onmessage = (event) => {
      setProcessedData(event.data);
    };
    
    worker.postMessage({
      type: 'processFinancialData',
      data: sampledData,
      metric,
      period
    });
    
    return () => {
      worker.terminate();
    };
  }, [isVisible, sampledData, metric, period]);
  
  return (
    <div ref={containerRef} className={`optimized-viz ${className || ''}`}>
      {renderChart()}
    </div>
  );
};

// Implementación del worker (financial-calculations.js)
self.onmessage = (event) => {
  const { type, data, metric, period } = event.data;
  
  if (type === 'processFinancialData') {
    // Cálculos pesados: tendencias, regresiones, pronósticos
    const result = performCalculations(data, metric, period);
    self.postMessage(result);
  }
};

// Función para muestreo adaptativo de series temporales
const downsampleTimeseries = (data, targetPoints) => {
  if (data.length <= targetPoints) return data;
  
  // Implementación del algoritmo Largest Triangle Three Buckets (LTTB)
  // que preserva características visuales importantes
  // ...código de implementación aquí...
};

// Sistema de caché para visualizaciones
const useVisualizationCache = () => {
  const cache = useRef(new Map());
  
  const getCachedVisualization = (key, data, renderFn) => {
    // Verificar si los datos han cambiado
    const cachedItem = cache.current.get(key);
    
    if (cachedItem) {
      const { cachedData, element } = cachedItem;
      
      // Comparar data con cachedData (comparación superficial o hash)
      if (
        JSON.stringify(data) === JSON.stringify(cachedData)
      ) {
        return element;
      }
    }
    
    // Renderizar y cachear
    const element = renderFn(data);
    cache.current.set(key, { cachedData: data, element });
    
    // Limitar tamaño del caché
    if (cache.current.size > 20) {
      // Eliminar item más antiguo
      const firstKey = cache.current.keys().next().value;
      cache.current.delete(firstKey);
    }
    
    return element;
  };
  
  return { getCachedVisualization };
};

// Componente de visualización con caché
const CachedVisualization = ({ cacheKey, data, render }) => {
  const { getCachedVisualization } = useVisualizationCache();
  
  return getCachedVisualization(cacheKey, data, render);
};
```

## 4. Wireframes de Alta Fidelidad

### 4.1 Dashboard Principal - Estado Base y Estados Alternativos

```
[Estado Base - con documentos recientes]
┌─────────────────────────────────────────────────────────────────┐
│ [Header con logo, navegación principal y perfil de usuario]     │
├─────────┬───────────────────────────────────────────────────────┤
│         │                                                       │
│         │ Dashboard                           [+ Nuevo Análisis]│
│         │                                                       │
│         │ ┌─────────────┐ ┌─────────────┐ ┌─────────────┐      │
│         │ │ Documentos  │ │ Análisis    │ │ Tiempo      │      │
│         │ │ Procesados  │ │ Completados │ │ Medio       │      │
│         │ │             │ │             │ │             │      │
│         │ │ 24          │ │ 18          │ │ 1.4 min     │      │
│         │ └─────────────┘ └─────────────┘ └─────────────┘      │
│ [Nav    │                                                       │
│  Side]  │ ┌─────────────────────────┐ ┌─────────────────────┐  │
│         │ │ Documentos Recientes    │ │ Actividad Reciente  │  │
│         │ │                         │ │                     │  │
│         │ │ • ACME Corp 10-K 2023   │ │ [Gráfico actividad] │  │
│         │ │   Completado - 2d atrás │ │                     │  │
│         │ │                         │ │                     │  │
│         │ │ • BizTech 10-Q Q3 2023  │ │                     │  │
│         │ │   Completado - 3d atrás │ │                     │  │
│         │ │                         │ │                     │  │
│         │ │ • FinGroup 10-K 2023    │ │                     │  │
│         │ │   Completado - 5d atrás │ │                     │  │
│         │ │                         │ │                     │  │
│         │ └─────────────────────────┘ └─────────────────────┘  │
│         │                                                       │
└─────────┴───────────────────────────────────────────────────────┘

[Estado Vacío - sin documentos]
┌─────────────────────────────────────────────────────────────────┐
│ [Header con logo, navegación principal y perfil de usuario]     │
├─────────┬───────────────────────────────────────────────────────┤
│         │                                                       │
│         │ Dashboard                           [+ Nuevo Análisis]│
│         │                                                       │
│         │ ┌─────────────────────────────────────────────────────┐
│         │ │                                                     │
│         │ │                   [Icono documento]                 │
│         │ │                                                     │
│         │ │            Aún no has procesado documentos          │
│         │ │                                                     │
│         │ │          Comienza cargando tu primer documento      │
│         │ │          financiero para recibir análisis detallados│
│         │ │                                                     │
│         │ │                [Cargar Documento]                   │
│         │ │                                                     │
│         │ └─────────────────────────────────────────────────────┘
│ [Nav    │                                                       │
│  Side]  │                                                       │
│         │                                                       │
│         │                                                       │
│         │                                                       │
│         │                                                       │
│         │                                                       │
│         │                                                       │
└─────────┴───────────────────────────────────────────────────────┘

[Estado Error - servicio no disponible]
┌─────────────────────────────────────────────────────────────────┐
│ [Header con logo, navegación principal y perfil de usuario]     │
├─────────┬───────────────────────────────────────────────────────┤
│         │                                                       │
│         │ Dashboard                           [+ Nuevo Análisis]│
│         │                                                       │
│         │ ┌─────────────────────────────────────────────────────┐
│         │ │                                                     │
│         │ │                 [Icono alerta rojo]                 │
│         │ │                                                     │
│         │ │      No se puede conectar con el servicio de datos  │
│         │ │                                                     │
│         │ │      Estamos experimentando problemas técnicos      │
│         │ │      temporales. Por favor, intenta nuevamente      │
│         │ │      en unos minutos.                               │
│         │ │                                                     │
│         │ │                   [Reintentar]                      │
│         │ │                                                     │
│         │ └─────────────────────────────────────────────────────┘
│ [Nav    │                                                       │
│  Side]  │                                                       │
│         │                                                       │
│         │                                                       │
│         │                                                       │
│         │                                                       │
│         │                                                       │
│         │                                                       │
└─────────┴───────────────────────────────────────────────────────┘
```

### 4.2 Carga de Documentos - Todos los Estados

```
[Estado Inicial - Carga]
┌─────────────────────────────────────────────────────────────────┐
│ Nuevo Análisis de Documento                                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│                                                                 │
│                                                                 │
│  ┌─────────────────────────────────────────────────────────────┐│
│  │                                                             ││
│  │                      [Icono documento]                      ││
│  │                                                             ││
│  │             Arrastra un documento 10-K o 10-Q               ││
│  │                o haz clic para seleccionar                  ││
│  │                                                             ││
│  │                  [Seleccionar Archivo]                      ││
│  │                                                             ││
│  │                                                             ││
│  │            Formatos soportados: PDF, DOCX, HTML             ││
│  │              Tamaño máximo: 50MB por archivo                ││
│  │                                                             ││
│  └─────────────────────────────────────────────────────────────┘│
│                                                                 │
│                                                                 │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘

[Estado Validación - Documento Seleccionado]
┌─────────────────────────────────────────────────────────────────┐
│ Nuevo Análisis de Documento                                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌─────────────────────────────────────────────────────────────┐│
│  │ Información del Documento                                   ││
│  │                                                             ││
│  │ [Vista previa miniatura]                                    ││
│  │                                                             ││
│  │ Nombre: ACME_Corporation_10K_2023.pdf                       ││
│  │ Tamaño: 3.2 MB                                              ││
│  │ Páginas: ~140                                               ││
│  │                                                             ││
│  │ [Cambiar archivo]                                           ││
│  └─────────────────────────────────────────────────────────────┘│
│                                                                 │
│  ┌─────────────────────────────────────────────────────────────┐│
│  │ Metadatos (opcional)                                        ││
│  │                                                             ││
│  │ Empresa:    [ACME Corporation____________]                  ││
│  │ Tipo:       [10-K ▼]                                        ││
│  │ Período:    [Anual 2023_______________]                     ││
│  │ Sector:     [Tecnología ▼]                                  ││
│  │                                                             ││
│  └─────────────────────────────────────────────────────────────┘│
│                                                                 │
│  [Cancelar]                  [Iniciar Procesamiento]            │
└─────────────────────────────────────────────────────────────────┘

[Estado Procesamiento - En progreso]
┌─────────────────────────────────────────────────────────────────┐
│ Procesando: ACME_Corporation_10K_2023.pdf                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Progreso total: 58%                           3:24 restantes   │
│  [====================-------------------------------]          │
│                                                                 │
│  ┌─────────────────────────────────────────────────────────────┐│
│  │ Fases de Procesamiento                                      ││
│  │                                                             ││
│  │ ● Validación de Formato            ✓ Completado             ││
│  │ ● Conversión PDF → Texto           ✓ Completado             ││
│  │ ● Chunking Estructural             ✓ Completado             ││
│  │ ● Chunking Semántico               ✓ Completado             ││
│  │ ● Detección de Tablas            ▶ En progreso (67%)        ││
│  │ ○ Extracción Financiera            Pendiente                ││
│  │ ○ Generación de Embeddings         Pendiente                ││
│  │ ○ Indexación Vectorial             Pendiente                ││
│  │ ○ Análisis Financiero              Pendiente                ││
│  │                                                             ││
│  │                                                             ││
│  └─────────────────────────────────────────────────────────────┘│
│                                                                 │
│  Recibirás una notificación cuando el proceso esté completo.    │
│  [Procesar en segundo plano]                                    │
└─────────────────────────────────────────────────────────────────┘

[Estado Error - Procesamiento fallido]
┌─────────────────────────────────────────────────────────────────┐
│ Error en Procesamiento: ACME_Corporation_10K_2023.pdf           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌─────────────────────────────────────────────────────────────┐│
│  │                                                             ││
│  │                   [Icono error circular]                    ││
│  │                                                             ││
│  │            Error en fase: Extracción Financiera             ││
│  │                                                             ││
│  │     No se pudieron extraer datos de las tablas principales. ││
│  │     El documento podría estar protegido o en un formato     ││
│  │     no estándar.                                            ││
│  │                                                             ││
│  │     Código de error: EXT-4032                               ││
│  │                                                             ││
│  │                                                             ││
│  └─────────────────────────────────────────────────────────────┘│
│                                                                 │
│  Recomendaciones:                                               │
│  • Verifica que el documento no esté protegido contra copia     │
│  • Intenta con una versión limpia del documento                 │
│  • Contacta a soporte técnico si el problema persiste           │
│                                                                 │
│  [Ver Log Detallado]  [Reintentar]  [Subir Nuevo Documento]     │
└─────────────────────────────────────────────────────────────────┘

[Estado Completado - Listo para análisis]
┌─────────────────────────────────────────────────────────────────┐
│ Procesamiento Completado: ACME_Corporation_10K_2023.pdf         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌─────────────────────────────────────────────────────────────┐│
│  │                                                             ││
│  │                 [Icono verificación verde]                  ││
│  │                                                             ││
│  │      ¡El documento ha sido procesado satisfactoriamente!    ││
│  │                                                             ││
│  │      Tiempo total: 1:42 minutos                             ││
│  │      140 páginas procesadas                                 ││
│  │      23 tablas financieras extraídas                        ││
│  │      89 métricas financieras identificadas                  ││
│  │                                                             ││
│  │      Precisión general estimada: 92%                        ││
│  │                                                             ││
│  └─────────────────────────────────────────────────────────────┘│
│                                                                 │
│  ¿Qué deseas hacer ahora?                                       │
│                                                                 │
│  [Iniciar otro análisis]       [Ver resultados del análisis]    │
└─────────────────────────────────────────────────────────────────┘
```

### 4.3 Vista de Análisis Dividida - Estados Principales

```
[Estado Completo - Vista dividida documento/análisis]
┌─────────────────────────────────────────────────────────────────┐
│ ACME Corp - Informe 10-K 2023          [Exportar] [Consultar ▼] │
├─────────────────────────────────────────────────────────────────┤
│ ┌──────────────────────────┐ ┌──────────────────────────────────┐
│ │ [Tabs documento]         │ │ [Tabs análisis]                  │
│ │                          │ │ Métricas | Ratios | Tendencias   │
│ │ [Visualización de        │ │                                  │
│ │  documento PDF con       │ │ Métricas Financieras Clave       │
│ │  tablas destacadas]      │ │ ┌──────────┐ ┌──────────┐        │
│ │                          │ │ │ Ingresos │ │ Margen   │        │
│ │                          │ │ │ $245.63M │ │ Bruto    │        │
│ │  ESTADO DE RESULTADOS    │ │ │ +11.9%   │ │ $98.85M  │        │
│ │  CONSOLIDADO             │ │ │          │ │ +16.0%   │        │
│ │                          │ │ └──────────┘ └──────────┘        │
│ │  ┌────────────────────┐  │ │ ┌──────────┐ ┌──────────┐        │
│ │  │Tabla financiera    │  │ │ │ Margen   │ │ % Margen │        │
│ │  │con datos de        │  │ │ │ Operativo│ │ Operativo│        │
│ │  │años fiscales       │  │ │ │ $57.09M  │ │ 23.2%    │        │
│ │  │2022 y 2023         │  │ │ │ +23.3%   │ │ +2.1%    │        │
│ │  │                    │  │ │ └──────────┘ └──────────┘        │
│ │  └────────────────────┘  │ │                                  │
│ │                          │ │ Análisis de Margen Operativo     │
│ │                          │ │ ┌────────────────────────────────┐
│ │                          │ │ │[Gráfico tendencia 5 años]      │
│ │                          │ │ │                                │
│ │                          │ │ │                                │
│ │                          │ │ └────────────────────────────────┘
│ │                          │ │                                  │
│ │                          │ │ El margen operativo ha aumentado │
│ │                          │ │ un 23.3% respecto al año anterior│
│ │                          │ │ debido principalmente a un       │
│ │                          │ │ incremento de ingresos con       │
│ │                          │ │ control efectivo de costos.      │
│ └──────────────────────────┘ └──────────────────────────────────┘
└─────────────────────────────────────────────────────────────────┘

[Estado Enfoque en Tabla - Visualización detallada]
┌─────────────────────────────────────────────────────────────────┐
│ ACME Corp - Informe 10-K 2023          [Exportar] [Consultar ▼] │
├─────────────────────────────────────────────────────────────────┤
│ ┌──────────────────────────┐ ┌──────────────────────────────────┐
│ │ [Tabs documento]         │ │ [Tabs análisis]                  │
│ │                          │ │ Métricas | Ratios | Tendencias   │
│ │ [Zoom en tabla           │ │                                  │
│ │  específica con          │ │ Detalles de Tabla Seleccionada   │
│ │  celdas destacadas]      │ │                                  │
│ │                          │ │ Estado de Resultados - Pág. 45   │
│ │                          │ │ Confianza de extracción: 96%     │
│ │  ESTADO DE RESULTADOS    │ │                                  │
│ │  CONSOLIDADO             │ │ ┌────────────────────────────────┐
│ │                          │ │ │[Visualización estructurada     │
│ │  ┌────────────────────┐  │ │ │de tabla con colores para       │
│ │  │Tabla financiera    │  │ │ │indicar cambios]                │
│ │  │con columnas de     │  │ │ │                                │
│ │  │2022-2023 destacadas│  │ │ │                                │
│ │  │y fila de margen    │  │ │ │                                │
│ │  │operativo resaltada │  │ │ │                                │
│ │  └────────────────────┘  │ │ └────────────────────────────────┘
│ │                          │ │                                  │
│ │                          │ │ Información Clave                │
│ │                          │ │ • Ingresos crecieron 11.9% YoY   │
│ │                          │ │ • Gastos operativos crecieron a  │
│ │                          │ │   menor ritmo que ingresos (7.4%)│
│ │                          │ │ • Mejora de eficiencia operativa │
│ │                          │ │   reflejada en incremento del    │
│ │                          │ │   margen operativo               │
│ │                          │ │                                  │
│ │                          │ │ [Ver información relacionada]    │
│ └──────────────────────────┘ └──────────────────────────────────┘
└─────────────────────────────────────────────────────────────────┘

[Estado Análisis Detallado - Ratios]
┌─────────────────────────────────────────────────────────────────┐
│ ACME Corp - Informe 10-K 2023          [Exportar] [Consultar ▼] │
├─────────────────────────────────────────────────────────────────┤
│ ┌──────────────────────────┐ ┌──────────────────────────────────┐
│ │ [Navegación de documento]│ │ [Tabs análisis]                  │
│ │ Visión general           │ │ Métricas | Ratios | Tendencias   │
│ │ MD&A                     │ │                                  │
│ │ Estados Financieros      │ │ Ratios Financieros               │
│ │ Notas                    │ │                                  │
│ │ Riesgos                  │ │ [Filtros: Categoría, Periodo]    │
│ │                          │ │                                  │
│ │ [Vista miniatura del     │ │ Liquidez                         │
│ │  documento con secciones]│ │ ┌─────────┐ ┌─────────┐ ┌────────┐
│ │                          │ │ │Ratio    │ │Ratio    │ │Ratio   │
│ │                          │ │ │Corriente│ │Rápido   │ │Efectivo│
│ │                          │ │ │         │ │         │ │        │
│ │                          │ │ │2.4      │ │1.8      │ │0.9     │
│ │                          │ │ │+0.3     │ │+0.2     │ │+0.1    │
│ │                          │ │ └─────────┘ └─────────┘ └────────┘
│ │                          │ │                                  │
│ │                          │ │ Rentabilidad                     │
│ │                          │ │ ┌─────────┐ ┌─────────┐ ┌────────┐
│ │                          │ │ │ROE      │ │ROA      │ │Margen  │
│ │                          │ │ │         │ │         │ │Neto    │
│ │                          │ │ │22.4%    │ │12.8%    │ │18.5%   │
│ │                          │ │ │+1.8%    │ │+1.2%    │ │+2.2%   │
│ │                          │ │ └─────────┘ └─────────┘ └────────┘
│ │                          │ │                                  │
│ │                          │ │ [Ver todos los ratios]           │
│ │                          │ │                                  │
│ └──────────────────────────┘ └──────────────────────────────────┘
└─────────────────────────────────────────────────────────────────┘

[Estado Análisis Comparativo - Tendencias]
┌─────────────────────────────────────────────────────────────────┐
│ ACME Corp - Informe 10-K 2023          [Exportar] [Consultar ▼] │
├─────────────────────────────────────────────────────────────────┤
│ ┌──────────────────────────┐ ┌──────────────────────────────────┐
│ │ [Timeline de documentos] │ │ [Tabs análisis]                  │
│ │                          │ │ Métricas | Ratios | Tendencias   │
│ │ ● 10-K 2023              │ │                                  │
│ │ ○ 10-Q Q3 2023           │ │ Análisis de Tendencias           │
│ │ ○ 10-Q Q2 2023           │ │                                  │
│ │ ○ 10-Q Q1 2023           │ │ [Filtros: Métrica, Periodo]      │
│ │ ○ 10-K 2022              │ │                                  │
│ │                          │ │ ┌────────────────────────────────┐
│ │ [Seleccionar período     │ │ │[Gráfico de línea mostrando     │
│ │  para comparación]       │ │ │tendencia de 8 períodos con     │
│ │                          │ │ │línea de benchmark sectorial]   │
│ │                          │ │ │                                │
│ │                          │ │ │                                │
│ │                          │ │ │                                │
│ │                          │ │ │                                │
│ │                          │ │ └────────────────────────────────┘
│ │                          │ │                                  │
│ │                          │ │ Comparativa Sectorial            │
│ │                          │ │ ┌────────────────────────────────┐
│ │                          │ │ │[Gráfico de barras comparando   │
│ │                          │ │ │misma métrica con competidores  │
│ │                          │ │ │y promedio del sector]          │
│ │                          │ │ │                                │
│ │                          │ │ │                                │
│ │                          │ │ └────────────────────────────────┘
│ │                          │ │                                  │
│ │                          │ │ [Exportar esta comparativa]      │
│ └──────────────────────────┘ └──────────────────────────────────┘
└─────────────────────────────────────────────────────────────────┘
```

### 4.4 Interfaz Conversacional - Estados y Variaciones

```
[Estado Conversación Inicial]
┌─────────────────────────────────────────────────────────────────┐
│ Consulta IA - ACME Corp 10-K 2023      [Exportar Conversación]  │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │                                                           │  │
│  │  [Icono FinDocs AI]                                       │  │
│  │                                                           │  │
│  │  Hola, soy tu asistente de análisis financiero. ¿En qué   │  │
│  │  puedo ayudarte con el informe 10-K 2023 de ACME Corp?    │  │
│  │                                                           │  │
│  │  Puedes preguntarme sobre:                                │  │
│  │  • Métricas financieras específicas                       │  │
│  │  • Tendencias y comparativas                              │  │
│  │  • Ratios y análisis                                      │  │
│  │  • Secciones específicas del documento                    │  │
│  │                                                           │  │
│  │                                         Hace 1 minuto     │  │
│  └───────────────────────────────────────────────────────────┘  │
│                                                                 │
│  Sugerencias:                                                   │
│  [¿Cuál es el margen operativo?] [Resumir cambios clave] [ROI]  │
│                                                                 │
│  ┌───────────────────────────────────────────────────┐ 📎  🎤   │
│  │ Escribe tu consulta...                            │          │
│  └───────────────────────────────────────────────────┘          │
└─────────────────────────────────────────────────────────────────┘

[Estado Conversación en Progreso]
┌─────────────────────────────────────────────────────────────────┐
│ Consulta IA - ACME Corp 10-K 2023      [Exportar Conversación]  │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │                                                           │  │
│  │  [Icono FinDocs AI]                                       │  │
│  │                                                           │  │
│  │  Hola, soy tu asistente de análisis financiero. ¿En qué   │  │
│  │  puedo ayudarte con el informe 10-K 2023 de ACME Corp?    │  │
│  │                                                           │  │
│  │                                         Hace 5 minutos    │  │
│  └───────────────────────────────────────────────────────────┘  │
│                                                                 │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │  [Usuario]                                                │  │
│  │                                                           │  │
│  │  ¿Cuál es el margen operativo del 2023 y cómo se         │  │
│  │  compara con el año anterior?                             │  │
│  │                                                           │  │
│  │                                         Hace 4 minutos    │  │
│  └───────────────────────────────────────────────────────────┘  │
│                                                                 │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │  [Icono FinDocs AI]                                       │  │
│  │                                                           │  │
│  │  El margen operativo para 2023 fue de $57.09 millones,    │  │
│  │  lo que representa un 23.2% sobre ingresos.               │  │
│  │                                                           │  │
│  │  Comparado con 2022, el margen operativo ha aumentado     │  │
│  │  un 23.3% (desde $46.29 millones), y el porcentaje        │  │
│  │  sobre ingresos ha mejorado 2.1 puntos porcentuales.      │  │
│  │                                                           │  │
│  │  Fuente: Página 45, Estado de Resultados                  │  │
│  │  [Ver en documento]                                       │  │
│  │                                                           │  │
│  │                                         Hace 3 minutos    │  │
│  └───────────────────────────────────────────────────────────┘  │
│                                                                 │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │  [Usuario]                                                │  │
│  │                                                           │  │
│  │  ¿Qué factores contribuyeron a esta mejora?               │  │
│  │                                                           │  │
│  │                                         Hace 2 minutos    │  │
│  └───────────────────────────────────────────────────────────┘  │
│                                                                 │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │  [Icono FinDocs AI]                                       │  │
│  │                                                           │  │
│  │  La mejora en el margen operativo se debe a varios        │  │
│  │  factores:                                                │  │
│  │                                                           │  │
│  │  1. Aumento de ingresos del 11.9% (de $219.45M a          │  │
│  │     $245.63M)                                             │  │
│  │  2. Control efectivo de gastos operativos, que solo       │  │
│  │     aumentaron un 7.4%                                    │  │
│  │  3. Mejor eficiencia en costos de ventas, con margen      │  │
│  │     bruto mejorado al 40.2% (vs 38.8% en 2022)            │  │
│  │                                                           │  │
│  │  El informe menciona específicamente en la sección MD&A   │  │
│  │  que las iniciativas de optimización operativa            │  │
│  │  implementadas en Q2 2023 resultaron en $3.2M de ahorros  │  │
│  │  anualizados.                                             │  │
│  │                                                           │  │
│  │  Fuentes: Página 45 (Estado de Resultados),               │  │
│  │  Página 23 (MD&A)                                         │  │
│  │  [Ver en documento]                                       │  │
│  │                                                           │  │
│  │                                         Hace 1 minuto     │  │
│  └───────────────────────────────────────────────────────────┘  │
│                                                                 │
│  Sugerencias:                                                   │
│  [Efecto en ROI] [Comparar con sector] [Tendencia 5 años]       │
│                                                                 │
│  ┌───────────────────────────────────────────────────┐ 📎  🎤   │
│  │ ¿Es sostenible esta tendencia?                    │          │
│  └───────────────────────────────────────────────────┘          │
└─────────────────────────────────────────────────────────────────┘

[Estado Generando Respuesta]
┌─────────────────────────────────────────────────────────────────┐
│ Consulta IA - ACME Corp 10-K 2023      [Exportar Conversación]  │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  [Mensajes previos]                                             │
│                                                                 │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │  [Usuario]                                                │  │
│  │                                                           │  │
│  │  ¿Es sostenible esta tendencia?                           │  │
│  │                                                           │  │
│  │                                         Hace 10 segundos  │  │
│  └───────────────────────────────────────────────────────────┘  │
│                                                                 │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │  [Icono FinDocs AI]                                       │  │
│  │                                                           │  │
│  │  Analizando sostenibilidad de la tendencia...             │  │
│  │                                                           │  │
│  │  • Consultando proyecciones de la gerencia                │  │
│  │  • Analizando factores macro y competitivos               │  │
│  │  • Revisando planes de expansión y desarrollo             │  │
│  │                                                           │  │
│  │  [Indicador de escritura animado]                         │  │
│  └───────────────────────────────────────────────────────────┘  │
│                                                                 │
│  [Cancelar consulta]                                            │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘

[Estado Error de Comprensión]
┌─────────────────────────────────────────────────────────────────┐
│ Consulta IA - ACME Corp 10-K 2023      [Exportar Conversación]  │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  [Mensajes previos]                                             │
│                                                                 │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │  [Usuario]                                                │  │
│  │                                                           │  │
│  │  Calcular la TCAC de los ingresos operativos ajustados    │  │
│  │  del segmento B2B                                         │  │
│  │                                                           │  │
│  │                                         Hace 10 segundos  │  │
│  └───────────────────────────────────────────────────────────┘  │
│                                                                 │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │  [Icono FinDocs AI]                                       │  │
│  │                                                           │  │
│  │  Lo siento, no encuentro información específica sobre     │  │
│  │  ingresos operativos ajustados del segmento B2B en el     │  │
│  │  documento 10-K 2023 de ACME Corp.                        │  │
│  │                                                           │  │
│  │  El documento presenta los ingresos por los siguientes    │  │
│  │  segmentos:                                               │  │
│  │  • Productos de consumo                                   │  │
│  │  • Servicios empresariales                                │  │
│  │  • Internacional                                          │  │
│  │                                                           │  │
│  │  ¿Te gustaría que calcule la TCAC para alguno de estos    │  │
│  │  segmentos en su lugar?                                   │  │
│  │                                                           │  │
│  │                                          Hace 5 segundos  │  │
│  └───────────────────────────────────────────────────────────┘  │
│                                                                 │
│  Sugerencias:                                                   │
│  [TCAC Servicios empresariales] [Ver segmentos] [Ingresos totales]│
│                                                                 │
│  ┌───────────────────────────────────────────────────┐ 📎  🎤   │
│  │ Escribe tu consulta...                            │          │
│  └───────────────────────────────────────────────────┘          │
└─────────────────────────────────────────────────────────────────┘

[Estado Visualización en Conversación]
┌─────────────────────────────────────────────────────────────────┐
│ Consulta IA - ACME Corp 10-K 2023      [Exportar Conversación]  │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  [Mensajes previos]                                             │
│                                                                 │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │  [Usuario]                                                │  │
│  │                                                           │  │
│  │  Muéstrame la tendencia del margen operativo de los      │  │
│  │  últimos 5 años                                           │  │
│  │                                                           │  │
│  │                                         Hace 30 segundos  │  │
│  └───────────────────────────────────────────────────────────┘  │
│                                                                 │
│  ┌───────────────────────────```
[Estado Visualización en Conversación - continuación]
┌─────────────────────────────────────────────────────────────────┐
│ Consulta IA - ACME Corp 10-K 2023      [Exportar Conversación]  │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  [Mensajes previos]                                             │
│                                                                 │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │  [Usuario]                                                │  │
│  │                                                           │  │
│  │  Muéstrame la tendencia del margen operativo de los      │  │
│  │  últimos 5 años                                           │  │
│  │                                                           │  │
│  │                                         Hace 30 segundos  │  │
│  └───────────────────────────────────────────────────────────┘  │
│                                                                 │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │  [Icono FinDocs AI]                                       │  │
│  │                                                           │  │
│  │  Aquí tienes la tendencia del margen operativo de ACME    │  │
│  │  Corp durante los últimos 5 años:                         │  │
│  │                                                           │  │
│  │  ┌─────────────────────────────────────────────────────┐  │  │
│  │  │                                                     │  │  │
│  │  │  [Gráfico de línea mostrando margen operativo       │  │  │
│  │  │   con tendencia creciente en los últimos 5 años]    │  │  │
│  │  │                                                     │  │  │
│  │  │                                                     │  │  │
│  │  │                                                     │  │  │
│  │  └─────────────────────────────────────────────────────┘  │  │
│  │                                                           │  │
│  │  Observaciones clave:                                     │  │
│  │  • Crecimiento constante durante los últimos 3 años      │  │
│  │  • Caída temporal en 2021 debido a la pandemia           │  │
│  │  • Fuerte recuperación y aceleración desde 2022          │  │
│  │                                                           │  │
│  │  Fuente: Estados financieros históricos, páginas 45-52   │  │
│  │  [Ver datos completos] [Comparar con sector]             │  │
│  │                                                           │  │
│  │                                         Hace 15 segundos  │  │
│  └───────────────────────────────────────────────────────────┘  │
│                                                                 │
│  Sugerencias:                                                   │
│  [Proyección próximos 2 años] [Factores de crecimiento] [ROI]   │
│                                                                 │
│  ┌───────────────────────────────────────────────────┐ 📎  🎤   │
│  │ Escribe tu consulta...                            │          │
│  └───────────────────────────────────────────────────┘          │
└─────────────────────────────────────────────────────────────────┘
```

### 4.5 Exportación de Análisis - Estados Completos

```
[Estado Selección de Formato]
┌─────────────────────────────────────────────────────────────────┐
│ Exportar Análisis: ACME Corp 10-K 2023                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Selecciona el formato de exportación:                          │
│                                                                 │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌──────────┐│
│  │ [Icono PDF] │  │[Icono Excel]│  │[Icono PPT]  │  │[Icono    ││
│  │             │  │             │  │             │  │JSON]     ││
│  │  PDF        │  │  Excel      │  │  PowerPoint │  │  JSON    ││
│  │             │  │             │  │             │  │          ││
│  │ Informe     │  │ Datos       │  │Presentación │  │ Datos    ││
│  │ completo con│  │ estructurad-│  │ ejecutiva   │  │ para API ││
│  │ análisis    │  │ os en hojas │  │ con gráficos│  │          ││
│  │             │  │             │  │             │  │          ││
│  │ [Selecciona-│  │             │  │             │  │          ││
│  │  do]        │  │             │  │             │  │          ││
│  └─────────────┘  └─────────────┘  └─────────────┘  └──────────┘│
│                                                                 │
│  Secciones a incluir:                                           │
│                                                                 │
│  ┌─────────────────────────────────────────────────────────────┐│
│  │ ☑ Resumen Ejecutivo                                        ││
│  │ ☑ Métricas Financieras Clave                               ││
│  │ ☑ Análisis de Tendencias                                   ││
│  │ ☑ Ratios Financieros                                       ││
│  │ ☐ Análisis Comparativo Sectorial                           ││
│  │ ☑ Visualizaciones y Gráficos                               ││
│  │ ☐ Datos Extraídos (Tablas Completas)                       ││
│  │ ☐ Transcripción de Conversación con IA                     ││
│  │ ☐ Texto Completo del Documento Original                    ││
│  └─────────────────────────────────────────────────────────────┘│
│                                                                 │
│  [Cancelar]                           [Continuar]               │
└─────────────────────────────────────────────────────────────────┘

[Estado Personalización de Reporte]
┌─────────────────────────────────────────────────────────────────┐
│ Exportar Análisis: ACME Corp 10-K 2023 - Personalización PDF    │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌─────────────────────────┐  ┌─────────────────────────────────┐
│  │                         │  │ Opciones de Formato             │
│  │ [Vista previa miniatura │  │                                 │
│  │  de portada de informe] │  │ ☑ Incluir portada              │
│  │                         │  │ ☑ Incluir tabla de contenidos   │
│  │                         │  │ ☑ Incluir resumen ejecutivo    │
│  │                         │  │ ☑ Incluir referencias a fuentes │
│  │                         │  │ ☐ Incluir notas al pie         │
│  │                         │  │ ☑ Incluir fecha de generación  │
│  │                         │  │                                 │
│  │                         │  │ Opciones de Estilo              │
│  │                         │  │                                 │
│  │                         │  │ Tema: [Profesional ▼]           │
│  │                         │  │ Paleta: [Azul corporativo ▼]    │
│  │                         │  │ Tamaño: [A4 ▼]                  │
│  │                         │  │                                 │
│  │                         │  │ Personalización                 │
│  │                         │  │                                 │
│  │                         │  │ ☑ Incluir logo de la empresa    │
│  │                         │  │ [Subir logo]                    │
│  │                         │  │                                 │
│  │                         │  │ Título: [Análisis Financiero...]│
│  │                         │  │ Subtítulo: [Informe 10-K 2023]  │
│  │                         │  │                                 │
│  └─────────────────────────┘  └─────────────────────────────────┘
│                                                                 │
│  [Volver]     [Vista Previa]                     [Exportar]     │
└─────────────────────────────────────────────────────────────────┘

[Estado Vista Previa]
┌─────────────────────────────────────────────────────────────────┐
│ Exportar Análisis: ACME Corp 10-K 2023 - Vista Previa           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌─────────────────────────────────────────────────────────────┐│
│  │                                                             ││
│  │ [Vista previa de documento PDF en pantalla completa,        ││
│  │  mostrando páginas en miniatura a la izquierda y            ││
│  │  visualización de página actual a la derecha]               ││
│  │                                                             ││
│  │                                                             ││
│  │                                                             ││
│  │                                                             ││
│  │                                                             ││
│  │                                                             ││
│  │                                                             ││
│  │                                                             ││
│  │                                                             ││
│  │                                                             ││
│  │                                                             ││
│  │                                                             ││
│  │                                                             ││
│  │                                                             ││
│  │                                                             ││
│  │                                                             ││
│  └─────────────────────────────────────────────────────────────┘│
│                                                                 │
│  [Volver a Editar]                                  [Exportar]  │
└─────────────────────────────────────────────────────────────────┘

[Estado Exportación Completada]
┌─────────────────────────────────────────────────────────────────┐
│ Exportar Análisis: ACME Corp 10-K 2023 - Completado             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌─────────────────────────────────────────────────────────────┐│
│  │                                                             ││
│  │                    [Icono verificación]                     ││
│  │                                                             ││
│  │             ¡Exportación completada con éxito!              ││
│  │                                                             ││
│  │  Se ha generado:                                            ││
│  │  • Análisis Financiero ACME Corp 10-K 2023.pdf              ││
│  │  • 24 páginas / 4.8 MB                                      ││
│  │  • 5 secciones / 12 visualizaciones                         ││
│  │                                                             ││
│  │                                                             ││
│  │       [Descargar]        [Compartir]        [Imprimir]      ││
│  │                                                             ││
│  │                                                             ││
│  │  Los análisis exportados también están disponibles en       ││
│  │  tu biblioteca de documentos para acceso futuro.            ││
│  │                                                             ││
│  │                                                             ││
│  └─────────────────────────────────────────────────────────────┘│
│                                                                 │
│  [Volver al Análisis]             [Generar Nuevo Análisis]      │
└─────────────────────────────────────────────────────────────────┘
```

## 5. Implementación Técnica de Redux

### 5.1 Manejo de Estado Asíncrono para Procesamiento de Documentos

```javascript
// Slice de Redux para procesamiento de documentos
import { createSlice, createAsyncThunk } from '@reduxjs/toolkit';
import { apiClient } from '../api/apiClient';

// Acción asíncrona para subir documento
export const uploadDocument = createAsyncThunk(
  'documents/upload',
  async ({ file, metadata }, { rejectWithValue }) => {
    try {
      const response = await apiClient.documents.upload(file, metadata);
      return response;
    } catch (error) {
      return rejectWithValue(error);
    }
  }
);

// Acción asíncrona para monitorear el procesamiento
export const fetchProcessingStatus = createAsyncThunk(
  'documents/fetchStatus',
  async (documentId, { rejectWithValue }) => {
    try {
      const response = await apiClient.documents.getProcessingStatus(documentId);
      return response;
    } catch (error) {
      return rejectWithValue(error);
    }
  }
);

// Slice para documentos y su procesamiento
const documentsSlice = createSlice({
  name: 'documents',
  initialState: {
    items: {},
    status: 'idle', // 'idle' | 'loading' | 'succeeded' | 'failed'
    error: null,
    currentDocumentId: null,
    processingDocumentId: null,
    processingProgress: null
  },
  reducers: {
    setCurrentDocument: (state, action) => {
      state.currentDocumentId = action.payload;
    },
    clearProcessingStatus: (state) => {
      state.processingDocumentId = null;
      state.processingProgress = null;
    }
  },
  extraReducers: (builder) => {
    builder
      // Subida de documento
      .addCase(uploadDocument.pending, (state) => {
        state.status = 'loading';
      })
      .addCase(uploadDocument.fulfilled, (state, action) => {
        state.status = 'succeeded';
        // Añadir documento al estado
        state.items[action.payload.id] = action.payload;
        // Iniciar monitoreo de procesamiento
        state.processingDocumentId = action.payload.id;
        state.processingProgress = {
          status: 'processing',
          currentStage: 'initializing',
          progress: 0,
          startedAt: new Date().toISOString(),
          estimatedCompletionTime: null
        };
      })
      .addCase(uploadDocument.rejected, (state, action) => {
        state.status = 'failed';
        state.error = action.payload || 'Error desconocido';
      })
      
      // Monitoreo de procesamiento
      .addCase(fetchProcessingStatus.fulfilled, (state, action) => {
        const { documentId, status } = action.payload;
        
        // Actualizar estado del documento
        if (state.items[documentId]) {
          state.items[documentId].status = status.status;
        }
        
        // Actualizar progreso de procesamiento
        if (state.processingDocumentId === documentId) {
          state.processingProgress = {
            status: status.status,
            currentStage: status.stage,
            progress: status.progress,
            startedAt: state.processingProgress.startedAt,
            estimatedCompletionTime: status.estimatedCompletionTime,
            completedStages: status.completedStages || []
          };
          
          // Si está completado, limpiar estado de procesamiento
          if (status.status === 'completed' || status.status === 'failed') {
            state.processingDocumentId = null;
          }
        }
      })
      .addCase(fetchProcessingStatus.rejected, (state, action) => {
        // Manejar error en monitoreo, pero continuar intentando
        if (state.processingProgress) {
          state.processingProgress.lastError = action.payload;
        }
      });
  }
});

export const { setCurrentDocument, clearProcessingStatus } = documentsSlice.actions;

// Selector para obtener el estado de procesamiento actual
export const selectProcessingStatus = (state) => {
  const { processingDocumentId, processingProgress } = state.documents;
  if (!processingDocumentId || !processingProgress) return null;
  
  return {
    documentId: processingDocumentId,
    ...processingProgress
  };
};

// Selector para determinar si un documento está siendo procesado
export const selectIsProcessing = (state, documentId) => {
  const document = state.documents.items[documentId];
  return document && document.status === 'processing';
};

export default documentsSlice.reducer;
```

### 5.2 Middleware para Monitoreo de Procesamiento

```javascript
// Middleware de Redux para monitoreo continuo de procesamiento
export const processingMonitorMiddleware = store => next => action => {
  const result = next(action);
  
  // Si se ha iniciado el procesamiento de un documento, configurar polling
  if (action.type === uploadDocument.fulfilled.type) {
    const documentId = action.payload.id;
    startProcessingMonitor(store, documentId);
  }
  
  // Si el procesamiento se ha completado en una actualización de estado
  if (action.type === fetchProcessingStatus.fulfilled.type) {
    const { status } = action.payload;
    if (status === 'completed' || status === 'failed') {
      // Detener el monitoreo
      stopProcessingMonitor(action.payload.documentId);
      
      // Mostrar notificación según resultado
      if (status === 'completed') {
        store.dispatch({
          type: 'ui/showNotification',
          payload: {
            type: 'success',
            message: 'Procesamiento de documento completado con éxito.',
            actions: [
              {
                label: 'Ver análisis',
                action: {
                  type: 'documents/setCurrentDocument',
                  payload: action.payload.documentId
                }
              }
            ]
          }
        });
      } else {
        store.dispatch({
          type: 'ui/showNotification',
          payload: {
            type: 'error',
            message: `Error en el procesamiento del documento: ${action.payload.error?.message || 'Error desconocido'}`,
            actions: [
              {
                label: 'Ver detalles',
                action: {
                  type: 'ui/openModal',
                  payload: {
                    modalType: 'processingError',
                    modalProps: { documentId: action.payload.documentId }
                  }
                }
              }
            ]
          }
        });
      }
    }
  }
  
  return result;
};

// Map para almacenar intervalos de monitoreo activos
const activeMonitors = new Map();

// Función para iniciar monitoreo
const startProcessingMonitor = (store, documentId) => {
  // Evitar duplicados
  if (activeMonitors.has(documentId)) {
    return;
  }
  
  // Configurar intervalo de polling
  const intervalId = setInterval(() => {
    store.dispatch(fetchProcessingStatus(documentId));
  }, 3000); // Actualizar cada 3 segundos
  
  // Almacenar referencia
  activeMonitors.set(documentId, intervalId);
};

// Función para detener monitoreo
const stopProcessingMonitor = (documentId) => {
  if (activeMonitors.has(documentId)) {
    clearInterval(activeMonitors.get(documentId));
    activeMonitors.delete(documentId);
  }
};
```

### 5.3 Componente de Monitoreo de Procesamiento

```jsx
// Componente de monitoreo de procesamiento con React y Redux
const ProcessingMonitor = ({ documentId }) => {
  const dispatch = useDispatch();
  const processingStatus = useSelector(state => selectProcessingStatus(state));
  const isCurrentDocumentProcessing = useSelector(state => {
    const status = selectProcessingStatus(state);
    return status && status.documentId === documentId;
  });
  
  // Mapeo de etapas a descripciones amigables
  const stageLabels = {
    'initializing': 'Inicialización',
    'validation': 'Validación de Formato',
    'conversion': 'Conversión PDF → Texto',
    'structural_chunking': 'Chunking Estructural',
    'semantic_chunking': 'Chunking Semántico',
    'table_detection': 'Detección de Tablas',
    'financial_extraction': 'Extracción Financiera',
    'embedding_generation': 'Generación de Embeddings',
    'vector_indexing': 'Indexación Vectorial',
    'financial_analysis': 'Análisis Financiero'
  };
  
  // Lista ordenada de etapas para visualización
  const orderedStages = [
    'initializing',
    'validation',
    'conversion',
    'structural_chunking',
    'semantic_chunking',
    'table_detection',
    'financial_extraction',
    'embedding_generation',
    'vector_indexing',
    'financial_analysis'
  ];
  
  // Manejar cancelación de procesamiento
  const handleCancelProcessing = () => {
    // Implementar lógica de cancelación
    dispatch(cancelProcessing(documentId));
  };
  
  // Si no hay procesamiento en curso
  if (!isCurrentDocumentProcessing) {
    return null;
  }
  
  // Calcular tiempo restante estimado
  const calculateRemainingTime = (estimatedCompletionTime) => {
    if (!estimatedCompletionTime) return 'Calculando...';
    
    const remaining = new Date(estimatedCompletionTime) - new Date();
    if (remaining <= 0) return 'Finalizando...';
    
    const minutes = Math.floor(remaining / 60000);
    const seconds = Math.floor((remaining % 60000) / 1000);
    
    if (minutes > 0) {
      return `${minutes}:${seconds.toString().padStart(2, '0')} restantes`;
    } else {
      return `${seconds} segundos restantes`;
    }
  };

  return (
    <div className="processing-monitor p-4 border rounded-lg bg-white shadow-sm">
      <div className="flex justify-between items-center mb-4">
        <h3 className="text-lg font-semibold">Procesando documento</h3>
        <div className="text-sm text-gray-500">
          {calculateRemainingTime(processingStatus.estimatedCompletionTime)}
        </div>
      </div>
      
      {/* Barra de progreso general */}
      <div className="mb-6">
        <div className="flex justify-between text-sm mb-1">
          <span>Progreso total</span>
          <span>{processingStatus.progress}%</span>
        </div>
        <div className="w-full bg-gray-200 rounded-full h-4">
          <div 
            className="bg-blue-600 h-4 rounded-full transition-all duration-300 ease-out"
            style={{ width: `${processingStatus.progress}%` }}
          />
        </div>
      </div>
      
      {/* Lista de etapas */}
      <div className="space-y-3 mb-6">
        <h4 className="font-medium text-sm text-gray-700">Fases de Procesamiento</h4>
        
        {orderedStages.map(stage => {
          // Determinar estado de esta etapa
          let status = 'pending';
          if (processingStatus.completedStages?.includes(stage)) {
            status = 'completed';
          } else if (processingStatus.currentStage === stage) {
            status = 'processing';
          }
          
          // Calcular progreso específico de la etapa actual
          const stageProgress = processingStatus.currentStage === stage 
            ? processingStatus.stageProgress || 0 
            : 0;
          
          return (
            <div key={stage} className="flex items-center">
              {/* Indicador de estado */}
              {status === 'completed' && (
                <div className="w-5 h-5 rounded-full bg-green-500 flex items-center justify-center mr-3">
                  <svg className="w-3 h-3 text-white" fill="none" stroke="currentColor" viewBox="0 0 24 24">
                    <path strokeLinecap="round" strokeLinejoin="round" strokeWidth="2" d="M5 13l4 4L19 7" />
                  </svg>
                </div>
              )}
              
              {status === 'processing' && (
                <div className="w-5 h-5 rounded-full border-2 border-blue-500 border-t-transparent animate-spin mr-3" />
              )}
              
              {status === 'pending' && (
                <div className="w-5 h-5 rounded-full bg-gray-300 mr-3" />
              )}
              
              {/* Nombre de la etapa */}
              <div className={`flex-grow ${status === 'pending' ? 'text-gray-500' : 'text-gray-900'}`}>
                {stageLabels[stage] || stage}
              </div>
              
              {/* Estado o progreso */}
              <div className="text-sm">
                {status === 'completed' && (
                  <span className="text-green-600">Completado</span>
                )}
                
                {status === 'processing' && (
                  <span className="text-blue-600">
                    {stageProgress > 0 ? `${stageProgress}%` : 'En progreso'}
                  </span>
                )}
                
                {status === 'pending' && (
                  <span className="text-gray-500">Pendiente</span>
                )}
              </div>
            </div>
          );
        })}
      </div>
      
      {/* Acciones */}
      <div className="flex justify-between">
        <div className="text-sm text-gray-500">
          Recibirás una notificación cuando el proceso esté completo.
        </div>
        <button 
          className="text-gray-700 hover:text-gray-900 text-sm font-medium"
          onClick={handleCancelProcessing}
        >
          Procesar en segundo plano
        </button>
      </div>
    </div>
  );
};
```

## 6. Especificaciones de Implementación para Visualizaciones

### 6.1 Componente de Tendencias Financieras con Optimización de Rendimiento

```jsx
// Componente optimizado para visualización de tendencias financieras
import React, { useEffect, useRef, useState, useMemo } from 'react';
import * as d3 from 'd3';
import { useResizeObserver } from '../../hooks/useResizeObserver';
import { formatCurrency, formatPercentage, formatPeriod } from '../../utils/formatters';

// Constantes de estilo para reutilización
const COLORS = {
  primary: '#0055B8',
  secondary: '#00A878',
  neutral: '#7F8C96',
  trend: {
    positive: '#10B981',
    negative: '#E63946',
    neutral: '#7F8C96'
  },
  grid: '#E9ECEF',
  background: '#F8F9FA'
};

const MARGINS = { top: 24, right: 40, bottom: 40, left: 50 };
const ANIMATION_DURATION = 500;
const TOOLTIP_DELAY = 200;

/**
 * Componente de Gráfico de Tendencias Financieras
 * 
 * @param {Object} props
 * @param {Array} props.data - Datos para visualizar en formato [{period: string, value: number, change: number}]
 * @param {Object} props.metric - Información de la métrica {name: string, unit: string}
 * @param {string} props.period - Tipo de periodo ('annual', 'quarterly')
 * @param {Object} props.benchmark - Datos de benchmark {value: number, label: string}
 * @param {number} props.height - Altura del gráfico en píxeles
 * @param {Function} props.onPointClick - Callback para clic en punto del gráfico
 * @param {string} props.className - Clases CSS adicionales
 * @param {boolean} props.showDataLabels - Mostrar etiquetas de datos
 */
const TrendChart = ({
  data,
  metric,
  period = 'annual',
  benchmark,
  height = 320,
  onPointClick,
  className = '',
  showDataLabels = true
}) => {
  const svgRef = useRef(null);
  const containerRef = useRef(null);
  const tooltipRef = useRef(null);
  const dimensions = useResizeObserver(containerRef);
  const [tooltip, setTooltip] = useState({ visible: false, x: 0, y: 0, content: null });
  
  // Procesamiento y validación de datos
  const validData = useMemo(() => {
    // Filtrar datos inválidos y ordenar por periodo
    if (!data || !Array.isArray(data)) return [];
    
    return [...data]
      .filter(d => d && d.period && d.value !== undefined && !isNaN(d.value))
      .sort((a, b) => new Date(a.period) - new Date(b.period));
  }, [data]);
  
  // No renderizar si no hay datos o dimensiones
  if (validData.length === 0) {
    return (
      <div 
        ref={containerRef} 
        className={`trend-chart ${className}`}
        style={{ height, display: 'flex', alignItems: 'center', justifyContent: 'center' }}
      >
        <div className="text-gray-500 text-sm">No hay datos suficientes para visualizar</div>
      </div>
    );
  }
  
  // Calcular cambios porcentuales si no están incluidos
  const enhancedData = useMemo(() => {
    return validData.map((d, i) => {
      if (i === 0 || d.change !== undefined) return d;
      
      const prev = validData[i - 1];
      const change = prev.value !== 0
        ? ((d.value - prev.value) / Math.abs(prev.value)) * 100
        : 0;
        
      return { ...d, change };
    });
  }, [validData]);
  
  // Efecto principal para crear y actualizar el gráfico
  useEffect(() => {
    if (!dimensions || !svgRef.current || enhancedData.length === 0) return;
    
    // Configurar dimensiones del área de dibujo
    const width = dimensions.width - MARGINS.left - MARGINS.right;
    const chartHeight = height - MARGINS.top - MARGINS.bottom;
    
    // Limpiar el SVG antes de dibujar
    const svg = d3.select(svgRef.current);
    svg.selectAll('*').remove();
    
    // Crear grupo principal con márgenes
    const g = svg.append('g')
      .attr('transform', `translate(${MARGINS.left},${MARGINS.top})`);
    
    // Crear escalas X e Y
    const xScale = d3.scaleTime()
      .domain(d3.extent(enhancedData, d => new Date(d.period)))
      .range([0, width])
      .nice();
    
    const yMin = d3.min(enhancedData, d => d.value) * 0.9; // 10% padding inferior
    const yMax = d3.max(enhancedData, d => d.value) * 1.1; // 10% padding superior
    
    const yScale = d3.scaleLinear()
      .domain([yMin < 0 ? yMin : 0, yMax]) // Asegurar que 0 esté incluido si hay valores positivos
      .range([chartHeight, 0])
      .nice();
    
    // Dibujar líneas de cuadrícula horizontales
    g.append('g')
      .attr('class', 'grid')
      .selectAll('line')
      .data(yScale.ticks(5))
      .enter()
      .append('line')
      .attr('x1', 0)
      .attr('x2', width)
      .attr('y1', d => yScale(d))
      .attr('y2', d => yScale(d))
      .attr('stroke', COLORS.grid)
      .attr('stroke-dasharray', '3,3')
      .attr('stroke-width', 1);
    
    // Línea de cero si el gráfico incluye valores negativos
    if (yMin < 0) {
      g.append('line')
        .attr('x1', 0)
        .attr('x2', width)
        .attr('y1', yScale(0))
        .attr('y2', yScale(0))
        .attr('stroke', COLORS.neutral)
        .attr('stroke-width', 1.5);
    }
    
    // Dibujar eje X
    const xAxis = g.append('g')
      .attr('transform', `translate(0,${chartHeight})`)
      .call(d3.axisBottom(xScale)
        .ticks(period === 'quarterly' ? d3.timeQuarter : d3.timeYear)
        .tickFormat(period === 'quarterly' ? 
          d => `Q${Math.floor(d.getMonth() / 3) + 1} ${d.getFullYear()}` : 
          d => d.getFullYear()
        )
      );
    
    // Estilizar eje X
    xAxis.selectAll('text')
      .style('font-size', '12px')
      .style('font-family', 'Inter, system-ui, sans-serif')
      .attr('dy', '0.7em');
    
    xAxis.selectAll('line')
      .style('stroke', COLORS.grid);
      
    xAxis.select('.domain')
      .style('stroke', COLORS.grid);
    
    // Dibujar eje Y con formato según tipo de métrica
    const yAxisTickFormat = (d) => {
      if (metric.unit === '%') return formatPercentage(d);
      if (metric.unit === '$') return formatCurrency(d, true); // formato compacto
      return d.toLocaleString();
    };
    
    const yAxis = g.append('g')
      .call(d3.axisLeft(yScale)
        .ticks(5)
        .tickFormat(yAxisTickFormat)
      );
    
    // Estilizar eje Y
    yAxis.selectAll('text')
      .style('font-size', '12px')
      .style('font-family', 'Inter, system-ui, sans-serif');
      
    yAxis.selectAll('line')
      .style('stroke', COLORS.grid);
      
    yAxis.select('.domain')
      .style('stroke', COLORS.grid);
    
    // Dibujar línea de benchmark si existe
    if (benchmark && benchmark.value) {
      g.append('line')
        .attr('x1', 0)
        .attr('x2', width)
        .attr('y1', yScale(benchmark.value))
        .attr('y2', yScale(benchmark.value))
        .attr('stroke', COLORS.neutral)
        .attr('stroke-width', 1.5)
        .attr('stroke-dasharray', '5,5')
        .attr('data-testid', 'benchmark-line');
      
      // Etiqueta de benchmark
      g.append('text')
        .attr('x', width)
        .attr('y', yScale(benchmark.value) - 5)
        .attr('text-anchor', 'end')
        .attr('fill', COLORS.neutral)
        .style('font-size', '11px')
        .style('font-family', 'Inter, system-ui, sans-serif')
        .text(benchmark.label || 'Benchmark');
    }
    
    // Crear generador de línea
    const line = d3.line()
      .x(d => xScale(new Date(d.period)))
      .y(d => yScale(d.value))
      .curve(d3.curveMonotoneX); // Curva suave
    
    // Dibujar línea principal con animación
    const path = g.append('path')
      .datum(enhancedData)
      .attr('fill', 'none')
      .attr('stroke', COLORS.primary)
      .attr('stroke-width', 2.5)
      .attr('d', line)
      .attr('data-testid', 'trend-line');
    
    // Animación de la línea
    const pathLength = path.node().getTotalLength();
    path
      .attr('stroke-dasharray', pathLength)
      .attr('stroke-dashoffset', pathLength)
      .transition()
      .duration(ANIMATION_DURATION)
      .ease(d3.easeLinear)
      .attr('stroke-dashoffset', 0);
    
    // Área bajo la curva (opcional, con animación)
    const area = d3.area()
      .x(d => xScale(new Date(d.period)))
      .y0(chartHeight)
      .y1(d => yScale(d.value))
      .curve(d3.curveMonotoneX);
    
    g.append('path')
      .datum(enhancedData)
      .attr('fill', COLORS.primary)
      .attr('fill-opacity', 0.1)
      .attr('d', area)
      .attr('data-testid', 'trend-area');
    
    // Dibujar puntos con animación retrasada
    const points = g.selectAll('.data-point')
      .data(enhancedData)
      .enter()
      .append('circle')
      .attr('class', 'data-point')
      .attr('cx', d => xScale(new Date(d.period)))
      .attr('cy', d => yScale(d.value))
      .attr('r', 0) // Empezar con radio 0 para animar
      .attr('fill', COLORS.primary)
      .attr('stroke', '#FFFFFF')
      .attr('stroke-width', 2)
      .attr('data-testid', 'data-point')
      .style('cursor', onPointClick ? 'pointer' : 'default')
      .on('click', (event, d) => {
        if (onPointClick) onPointClick(d, enhancedData.indexOf(d));
      });
      
    // Animar puntos
    points.transition()
      .delay((d, i) => ANIMATION_DURATION + (i * 50))
      .duration(200)
      .attr('r', 5);
    
    // Etiquetas de datos si están habilitadas
    if (showDataLabels) {
      g.selectAll('.data-label')
        .data(enhancedData)
        .enter()
        .append('text')
        .attr('class', 'data-label')
        .attr('x', d => xScale(new Date(d.period)))
        .attr('y', d => yScale(d.value) - 12)
        .attr('text-anchor', 'middle')
        .style('font-family', 'Roboto Mono, monospace')
        .style('font-size', '11px')
        .style('font-weight', '500')
        .style('fill', COLORS.primary)
        .style('opacity', 0)
        .text(d => {
          if (metric.unit === '%') return formatPercentage(d.value);
          if (metric.unit === '$') return formatCurrency(d.value, true);
          return d.value.toLocaleString();
        })
        .transition()
        .delay((d, i) => ANIMATION_DURATION + (i * 50) + 100)
        .duration(200)
        .style('opacity', 1);
    }
    
    // Crear área de interacción para tooltips (más grande que los puntos)
    g.selectAll('.interaction-area')
      .data(enhancedData)
      .enter()
      .append('circle')
      .attr('class', 'interaction-area')
      .attr('cx', d => xScale(new Date(d.period)))
      .attr('cy', d => yScale(d.value))
      .attr('r', 15) // Radio más grande para facilitar interacción
      .attr('fill', 'transparent')
      .style('cursor', 'pointer')
      .on('mouseenter', (event, d) => {
        // Resaltar punto
        d3.select(points.nodes()[enhancedData.indexOf(d)])
          .transition()
          .duration(100)
          .attr('r', 7);
        
        // Mostrar tooltip
        const periodFormatted = formatPeriod(d.period, period);
        const valueFormatted = metric.unit === '%' 
          ? formatPercentage(d.value) 
          : metric.unit === '$' 
            ? formatCurrency(d.value) 
            : d.value.toLocaleString();
            
        const content = `
          <div class="font-medium">${periodFormatted}</div>
          <div>${metric.name}: ${valueFormatted}</div>
          ${d.change !== undefined ? `<div class="${d.change >= 0 ? 'text-green-600' : 'text-red-600'}">
            ${d.change >= 0 ? '▲' : '▼'} ${Math.abs(d.change).toFixed(1)}%
          </div>` : ''}
        `;
        
        setTooltip({
          visible: true,
          x: event.pageX,
          y: event.pageY - 28,
          content
        });
      })
      .on('mouseleave', (event, d) => {
        // Restaurar punto
        d3.select(points.nodes()[enhancedData.indexOf(d)])
          .transition()
          .duration(100)
          .attr('r', 5);
        
        // Ocultar tooltip
        setTooltip({ ...tooltip, visible: false });
      })
      .on('click', (event, d) => {
        if (onPointClick) onPointClick(d, enhancedData.indexOf(d));
      });

  }, [dimensions, enhancedData, metric, period, benchmark, height, showDataLabels, onPointClick]);
  
  return (
    <div ref={containerRef} className={`trend-chart ${className}`} style={{ position: 'relative' }}>
      {/* Título del gráfico */}
      <div className="text-sm font-medium mb-2">{metric?.name || 'Tendencia'}</div>
      
      {/* SVG principal */}
      <svg 
        ref={svgRef} 
        width="100%" 
        height={height}
        aria-label={`Gráfico de tendencia de ${metric?.name || 'métrica financiera'}`}
        role="img"
      />
      
      {/* Tooltip */}
      {tooltip.visible && (
        <div 
          ref={tooltipRef}
          className="absolute z-10 bg-white p-2 rounded shadow-lg border border-gray-200 text-xs"
          style={{ 
            left: `${tooltip.x}px`, 
            top: `${tooltip.y}px`,
            transform: 'translate(-50%, -100%)',
            pointerEvents: 'none'
          }}
          dangerouslySetInnerHTML={{ __html: tooltip.content }}
        />
      )}
    </div>
  );
};

// Memoizar el componente para evitar re-renderizados innecesarios
export default React.memo(TrendChart);
```

### 6.2 Hook para Monitoreo Eficiente de Dimensiones

```jsx
// Hook para observar cambios de tamaño en elementos DOM
import { useEffect, useState } from 'react';

/**
 * Hook que utiliza ResizeObserver para monitorear cambios de tamaño
 * en un elemento de referencia.
 * 
 * @param {React.RefObject} ref - Referencia al elemento a observar
 * @param {number} debounceMs - Tiempo de debounce en milisegundos
 * @returns {Object|null} Dimensiones actualizadas {width, height} o null
 */
export const useResizeObserver = (ref, debounceMs = 100) => {
  const [dimensions, setDimensions] = useState(null);
  const [debouncedDimensions, setDebouncedDimensions] = useState(null);
  
  useEffect(() => {
    const element = ref.current;
    if (!element) return;
    
    // Capturar dimensiones iniciales
    setDimensions({
      width: element.clientWidth,
      height: element.clientHeight
    });
    
    // Configurar observador
    const resizeObserver = new ResizeObserver(entries => {
      if (!entries || !entries[0]) return;
      
      const { width, height } = entries[0].contentRect;
      
      // Actualizar dimensiones inmediatamente para animaciones fluidas
      setDimensions({ width, height });
    });
    
    resizeObserver.observe(element);
    
    return () => {
      if (element) resizeObserver.unobserve(element);
    };
  }, [ref]);
  
  // Aplicar debounce para evitar renderizados excesivos
  useEffect(() => {
    if (!dimensions) return;
    
    const timerId = setTimeout(() => {
      setDebouncedDimensions(dimensions);
    }, debounceMs);
    
    return () => {
      clearTimeout(timerId);
    };
  }, [dimensions, debounceMs]);
  
  // Devolver el valor debounced para la mayoría de los casos,
  // pero si aún no existe, devolver las dimensiones instantáneas
  return debouncedDimensions || dimensions;
};

// Hook adicional para manejo de elementos y dimensiones específicamente para visualizaciones
export const useVisualizationElement = (initialHeight = 320) => {
  const containerRef = useRef(null);
  const dimensions = useResizeObserver(containerRef);
  const [height, setHeight] = useState(initialHeight);
  
  // Exponer métodos para trabajar con dimensiones
  const resize = (newHeight) => {
    setHeight(newHeight);
  };
  
  // Cálculo de dimensiones efectivas optimizado para visualizaciones
  const effectiveDimensions = useMemo(() => {
    if (!dimensions) return { width: 0, height, aspectRatio: 0 };
    
    const { width } = dimensions;
    const aspectRatio = width / height;
    
    return { width, height, aspectRatio };
  }, [dimensions, height]);
  
  return {
    containerRef,
    dimensions: effectiveDimensions,
    resize
  };
};
```

### 6.3 Componente de Tabla Financiera con Destacado Interactivo

```jsx
// Componente de tabla financiera con interacción y resaltado
import React, { useState, useMemo } from 'react';
import { formatCurrency, formatPercentage } from '../../utils/formatters';
import { calculateGrowth, calculateCAGR } from '../../utils/financialCalculations';

/**
 * Componente de Tabla Financiera Interactiva
 * 
 * @param {Object} props
 * @param {Array} props.data - Datos de la tabla en formato apropiado
 * @param {Array} props.columns - Definiciones de columnas
 * @param {Array} props.metrics - Métricas a mostrar
 * @param {Array} props.periods - Periodos a mostrar
 * @param {string} props.highlightedMetric - Métrica resaltada
 * @param {Function} props.onCellClick - Callback para clic en celda
 * @param {string} props.className - Clases CSS adicionales
 */
const FinancialTable = ({
  data,
  columns,
  metrics,
  periods,
  highlightedMetric,
  onCellClick,
  className = ''
}) => {
  const [hoveredRow, setHoveredRow] = useState(null);
  const [hoveredColumn, setHoveredColumn] = useState(null);
  
  // Procesamiento de datos para formato de tabla
  const tableData = useMemo(() => {
    if (!data || !metrics || !periods) return [];
    
    return metrics.map(metric => {
      const row = {
        id: metric.id,
        name: metric.name,
        unit: metric.unit
      };
      
      // Añadir valores para cada periodo
      periods.forEach(period => {
        const value = data.find(d => 
          d.metric === metric.id && d.period === period
        )?.value;
        
        row[period] = value;
      });
      
      // Calcular cambios si hay más de un periodo
      if (periods.length > 1) {
        const firstPeriod = periods[0];
        const lastPeriod = periods[periods.length - 1];
        
        // Crecimiento entre último y primer periodo
        if (row[firstPeriod] !== undefined && row[lastPeriod] !== undefined) {
          row.growth = calculateGrowth(row[firstPeriod], row[lastPeriod]);
          
          // CAGR si hay más de 2 años
          if (periods.length > 2) {
            const yearCount = periods.length - 1; // Asumiendo periodos anuales
            row.cagr = calculateCAGR(row[firstPeriod], row[lastPeriod], yearCount);
          }
        }
      }
      
      return row;
    });
  }, [data, metrics, periods]);
  
  // Formatear valor según el tipo de unidad
  const formatCellValue = (value, unit) => {
    if (value === undefined || value === null) return '-';
    
    if (unit === '%') return formatPercentage(value);
    if (unit === '$') return formatCurrency(value);
    return value.toLocaleString();
  };
  
  // Determinar color para celdas con cambio porcentual
  const getChangeColor = (value) => {
    if (value === undefined || value === null) return '';
    
    if (value > 0) return 'text-green-600';
    if (value < 0) return 'text-red-600';
    return 'text-gray-500';
  };
  
  // Determinar si una celda debe ser resaltada
  const isHighlighted = (metricId, period) => {
    return metricId === highlightedMetric || period === hoveredColumn || metricId === hoveredRow;
  };
  
  return (
    <div className={`financial-table-container overflow-x-auto ${className}`}>
      <table className="w-full text-sm border-collapse">
        <thead>
          <tr className="bg-gray-50">
            <th className="px-4 py-2 border-b border-gray-300 text-left font-medium">
              Métrica
            </th>
            
            {periods.map(period => (
              <th 
                key={period}
                className={`px-4 py-2 border-b border-gray-300 text-right font-medium ${
                  hoveredColumn === period ? 'bg-blue-50' : ''
                }`}
                onMouseEnter={() => setHoveredColumn(period)}
                onMouseLeave={() => setHoveredColumn(null)}
              >
                {period}
              </th>
            ))}
            
            {periods.length > 1 && (
              <th className="px-4 py-2 border-b border-gray-300 text-right font-medium">
                Cambio
              </th>
            )}
            
            {periods.length > 2 && (
              <th className="px-4 py-2 border-b border-gray-300 text-right font-medium">
                CAGR
              </th>
            )}
          </tr>
        </thead>
        
        <tbody>
          {tableData.map(row => (
            <tr 
              key={row.id}
              className={`${
                isHighlighted(row.id, null) ? 'bg-blue-50' : 
                tableData.indexOf(row) % 2 === 0 ? 'bg-white' : 'bg-gray-50'
              } hover:bg-blue-50 transition-colors duration-150`}
              onMouseEnter={() => setHoveredRow(row.id)}
              onMouseLeave={() => setHoveredRow(null)}
              data-testid={`financial-table-row-${row.id}`}
            >
              <td className="px-4 py-2 border-b border-gray-200 font-medium">
                {row.name}
              </td>
              
              {periods.map(period => (
                <td 
                  key={`${row.id}-${period}`}
                  className={`px-4 py-2 border-b border-gray-200 text-right ${
                    isHighlighted(row.id, period) ? 'bg-blue-100' : ''
                  }`}
                  onClick={() => onCellClick && onCellClick(row.id, period, row[period])}
                  style={{ cursor: onCellClick ? 'pointer' : 'default' }}
                  data-testid={`cell-${row.id}-${period}`}
                >
                  {formatCellValue(row[period], row.unit)}
                </td>
              ))}
              
              {periods.length > 1 && (
                <td className={`px-4 py-2 border-b border-gray-200 text-right ${getChangeColor(row.growth)}`}>
                  {row.growth !== undefined ? `${row.growth >= 0 ? '+' : ''}${formatPercentage(row.growth)}` : '-'}
                </td>
              )}
              
              {periods.length > 2 && (
                <td className={`px-4 py-2 border-b border-gray-200 text-right ${getChangeColor(row.cagr)}`}>
                  {row.cagr !== undefined ? formatPercentage(row.cagr) : '-'}
                </td>
              )}
            </tr>
          ))}
        </tbody>
      </table>
    </div>
  );
};

// Memoizar el componente para evitar re-renderizados innecesarios
export default React.memo(FinancialTable);
```

## 7. Guía de Estilos Completa

```javascript
// Definición de sistema de diseño en formato de tokens
export const designTokens = {
  // Colores principales
  colors: {
    primary: {
      50: '#E6F0F9',
      100: '#CCE0F3',
      200: '#99C2E7',
      300: '#66A3DB',
      400: '#3385CF',
      500: '#0055B8', // Color principal
      600: '#0044A6',
      700: '#003380',
      800: '#00225A',
      900: '#001133'
    },
    secondary: {
      50: '#E6F7F2',
      100: '#CCEFE5',
      200: '#99DFCC',
      300: '#66CFB2',
      400: '#33BF99',
      500: '#00A878', // Color secundario
      600: '#00966A',
      700: '#007452',
      800: '#004F39',
      900: '#00271D'
    },
    neutral: {
      50: '#F8F9FA',
      100: '#E9ECEF',
      200: '#DEE2E6',
      300: '#CED4DA',
      400: '#ADB5BD',
      500: '#6C757D',
      600: '#495057',
      700: '#343A40',
      800: '#212529',
      900: '#121416'
    },
    success: '#10B981',
    warning: '#F59E0B',
    error: '#E63946',
    info: '#3B82F6'
  },
  
  // Tipografía
  typography: {
    fontFamily: {
      sans: 'Inter, system-ui, -apple-system, sans-serif',
      mono: 'Roboto Mono, SFMono-Regular, Menlo, monospace'
    },
    fontWeight: {
      regular: 400,
      medium: 500,
      semibold: 600,
      bold: 700
    },
    fontSize: {
      xs: '0.75rem',     // 12px
      sm: '0.875rem',    // 14px
      base: '1rem',      // 16px
      lg: '1.125rem',    // 18px
      xl: '1.25rem',     // 20px
      '2xl': '1.5rem',   // 24px
      '3xl': '1.875rem', // 30px
      '4xl': '2.25rem'   // 36px
    },
    lineHeight: {
      none: 1,
      tight: 1.25,
      normal: 1.5,
      relaxed: 1.75
    }
  },
  
  // Espaciado (basado en 8px)
  spacing: {
    0: '0',
    1: '0.25rem',  // 4px
    2: '0.5rem',   // 8px
    3: '0.75rem',  // 12px
    4: '1rem',     // 16px
    5: '1.25rem',  // 20px
    6: '1.5rem',   // 24px
    8: '2rem',     // 32px
    10: '2.5rem',  // 40px
    12: '3rem',    // 48px
    16: '4rem',    // 64px
    20: '5rem',    // 80px
    24: '6rem'     // 96px
  },
  
  // Bordes y Radios
  borders: {
    radius: {
      none: '0',
      sm: '0.125rem',    // 2px
      DEFAULT: '0.25rem', // 4px
      md: '0.375rem',    // 6px
      lg: '0.5rem',      // 8px
      xl: '0.75rem',     // 12px
      '2xl': '1rem',     // 16px
      full: '9999px'
    },
    width: {
      none: '0',
      thin: '1px',
      thick: '2px',
      heavy: '4px'
    }
  },
  
  // Sombras
  shadows: {
    none: 'none',
    sm: '0 1px 2px 0 rgba(0, 0, 0, 0.05)',
    DEFAULT: '0 1px 3px 0 rgba(0, 0, 0, 0.1), 0 1px 2px 0 rgba(0, 0, 0, 0.06)',
    md: '0 4px 6px -1px rgba(0, 0, 0, 0.1), 0 2px 4px -1px rgba(0, 0, 0, 0.06)',
    lg: '0 10px 15px -3px rgba(0, 0, 0, 0.1), 0 4px 6px -2px rgba(0, 0, 0, 0.05)',
    xl: '0 20px 25px -5px rgba(0, 0, 0, 0.1), 0 10px 10px -5px rgba(0, 0, 0, 0.04)'
  },
  
  // Transiciones
  transitions: {
    duration: {
      fast: '150ms',
      normal: '300ms',
      slow: '500ms'
    },
    timing: {
      ease: 'ease',
      linear: 'linear',
      easeIn: 'cubic-bezier(0.4, 0, 1, 1)',
      easeOut: 'cubic-bezier(0, 0, 0.2, 1)',
      easeInOut: 'cubic-bezier(0.4, 0, 0.2, 1)'
    }
  },
  
  // Breakpoints responsivos
  screens: {
    xs: '480px',
    sm: '640px',
    md: '768px',
    lg: '1024px',
    xl: '1280px',
    '2xl': '1536px'
  },
  
  // Z-índices
  zIndex: {
    auto: 'auto',
    0: '0',
    10: '10',
    20: '20',
    30: '30',
    40: '40',
    50: '50',
    dropdown: '1000',
    sticky: '1020',
    fixed: '1030',
    modalBackdrop: '1040',
    modal: '1050',
    popover: '1060',
    tooltip: '1070'
  }
};

// Exportar componentes de diseño (implementación con CSS-in-JS o CSS Modules)
export const designComponents = {
  /**
   * Botón principal de la aplicación
   */
  button: {
    base: `
      inline-flex items-center justify-center
      rounded-md
      font-medium
      transition-colors duration-150
      focus:outline-none focus:ring-2 focus:ring-offset-2
      disabled:opacity-50 disabled:cursor-not-allowed
    `,
    sizes: {
      sm: 'px-3 py-1.5 text-xs',
      md: 'px-4 py-2 text-sm',
      lg: 'px-5 py-2.5 text-base'
    },
    variants: {
      primary: `
        bg-primary-500 text-white
        hover:bg-primary-600
        focus:ring-primary-500
      `,
      secondary: `
        bg-secondary-500 text-white
        hover:bg-secondary-600
        focus:ring-secondary-500
      `,
      outline: `
        bg-transparent border border-primary-500 text-primary-500
        hover:bg-primary-50
        focus:ring-primary-500
      `,
      ghost: `
        bg-transparent text-primary-500
        hover:bg-primary-50
        focus:ring-primary-500
      `
    }
  },
  
  /**
   * Tarjeta contenedora
   */
  card: {
    base: `
      bg-white
      rounded-lg
      border border-neutral-200
      shadow-sm
      overflow-hidden
    `,
    variants: {
      flat: '',
      raised: 'shadow-md',
      interactive: 'cursor-pointer hover:shadow-md transition-shadow duration-normal'
    },
    header: `
      px-6 py-4
      border-b border-neutral-200
      flex items-center justify-between
    `,
    body: `
      p-6
    `,
    footer: `
      px-6 py-4
      border-t border-neutral-200
      bg-neutral-50
    `
  },
  
  /**
   * Campo de formulario
   */
  formField: {
    base: `
      flex flex-col space-y-1
    `,
    label: `
      text-sm font-medium text-neutral-700
    `,
    input: `
      block w-full
      px-4 py-2
      border border-neutral-300 rounded-md
      text-neutral-700
      placeholder:text-neutral-400
      focus:outline-none focus:ring-2 focus:ring-primary-500 focus:border-primary-500
      disabled:opacity-50 disabled:bg-neutral-100
    `,
    error: `
      text-xs font-medium text-error
    `,
    hint: `
      text-xs text-neutral-500
    `
  },
  
  /**
   * Componentes de navegación
   */
  navigation: {
    main: `
      bg-white
      border-b border-neutral-200
      shadow-sm
    `,
    sidebar: `
      bg-white
      border-r border-neutral-200
      h-full
    `,
    sidebarItem: {
      base: `
        flex items-center space-x-2
        px-4 py-2
        text-sm font-medium
        rounded-md
        cursor-pointer
      `,
      active: `
        bg-primary-50 text-primary-600
      `,
      inactive: `
        text-neutral-700
        hover:bg-neutral-100
      `
    },
    tabs: {
      container: `
        flex space-x-1
        border-b border-neutral-200
      `,
      tab: {
        base: `
          px-4 py-2
          text-sm font-medium
          cursor-pointer
        `,
        active: `
          text-primary-600
          border-b-2 border-primary-500
        `,
        inactive: `
          text-neutral-500
          hover:text-neutral-700 hover:border-b-2 hover:border-neutral-300
        `
      }
    }
  },
  
  /**
   * Indicadores de progreso
   */
  progress: {
    container: `
      bg-neutral-200
      rounded-full
      overflow-hidden
    `,
    bar: `
      h-full
      bg-primary-500
      transition-all duration-300 ease-out
    `,
    steps: {
      container: `
        flex items-center
      `,
      step: {
        base: `
          flex flex-col items-center
        `,
        circle: {
          base: `
            w-8 h-8
            rounded-full
            flex items-center justify-center
            text-sm font-medium
            border-2
          `,
          active: `
            bg-primary-500 text-white
            border-primary-500
          `,
          completed: `
            bg-primary-100 text-primary-700
            border-primary-500
          `,
          pending: `
            bg-white text-neutral-400
            border-neutral-300
          `
        },
        label: `
          mt-2
          text-xs font-medium
        `,
        connector: `
          flex-grow
          h-0.5
          mx-2
        `
      }
    }
  },
  
  /**
   * Notificaciones y alertas
   */
  alert: {
    base: `
      p-4
      rounded-md
      border
      flex items-start
    `,
    variants: {
      success: `
        bg-green-50
        border-green-200
        text-green-800
      `,
      error: `
        bg-red-50
        border-red-200
        text-red-800
      `,
      warning: `
        bg-yellow-50
        border-yellow-200
        text-yellow-800
      `,
      info: `
        bg-blue-50
        border-blue-200
        text-blue-800
      `
    },
    icon: `
      mr-3 mt-0.5
    `,
    content: `
      flex-grow
    `,
    title: `
      text-sm font-semibold
      mb-1
    `,
    description: `
      text-sm
    `,
    close: `
      ml-3
      text-neutral-400
      hover:text-neutral-600
      cursor-pointer
    `
  },
  
  /**
   * Componentes específicos para finanzas
   */
  finance: {
    metricCard: {
      base: `
        p-4
        bg-white
        rounded-lg
        border border-neutral-200
        shadow-sm
      `,
      label: `
        text-xs text-neutral-500
        mb-1
      `,
      value: `
        text-lg font-semibold
      `,
      change: {
        base: `
          text-xs font-medium
          ml-2
        `,
        positive: `
          text-success
        `,
        negative: `
          text-error
        `,
        neutral: `
          text-neutral-500
        `
      },
      confidence: {
        base: `
          text-xs
          mt-2
          flex items-center
        `,
        high: `
          text-success
        `,
        medium: `
          text-warning
        `,
        low: `
          text-error
        `,
        indicator: `
          w-2 h-2
          rounded-full
          mr-1
        `
      }
    },
    dataTable: {
      base: `
        w-full
        border-collapse
        text-sm
      `,
      header: `
        bg-neutral-50
        text-left
        font-medium
        text-neutral-700
      `,
      headerCell: `
        px-4 py-2
        border-b border-neutral-300
      `,
      row: {
        base: `
          border-b border-neutral-200
          transition-colors duration-150
        `,
        even: `
          bg-white
        `,
        odd: `
          bg-neutral-50
        `,
        hover: `
          hover:bg-blue-50
        `
      },
      cell: `
        px-4 py-2
      `,
      highlight: `
        bg-blue-100
      `
    }
  },
  
  // Componentes adicionales según necesidad
};

// Exportar utilidad para combinar clases condicionales
export const classNames = (...classes) => {
  return classes.filter(Boolean).join(' ');
};
```

## 8. Recomendaciones Finales para Implementación

1. **Enfoque de Implementación Recomendado**:
   - Implementar primero el flujo completo de procesamiento (MVP mínimo)
   - Continuar con el módulo de visualización y análisis
   - Finalizar con el módulo conversacional
   - Implementar exportación como último módulo

2. **Estrategia de Testing**:
   - Implementar pruebas unitarias para funciones de utilidad financiera
   - Pruebas de integración para flujos de API
   - Pruebas de renderizado para componentes visuales
   - End-to-end testing para flujos críticos (procesamiento, análisis)

3. **Plan de Desarrollo Sugerido (6 semanas)**:

   **Semana 1: Configuración y Flujo Básico**
   - Configuración del proyecto (3 días)
   - Implementación de autenticación (1 día)
   - Dashboard y navegación básica (1 día)

   **Semana 2: Procesamiento de Documentos**
   - Implementación de carga y validación (2 días)
   - Monitor de progreso de procesamiento (2 días)
   - Estados de error y notificaciones (1 día)

   **Semana 3: Visualización de Documentos**
   - Vista dividida documento/análisis (2 días)
   - Navegación del documento (1 día)
   - Visualización de tablas financieras (2 días)

   **Semana 4: Visualizaciones Financieras**
   - Implementación de gráficos base (2 días)
   - Tablas interactivas (1 día)
   - Comparativas y tendencias (2 días)

   **Semana 5: Interfaz Conversacional**
   - Implementación de chat básico (2 días)
   - Integración con contexto de documento (2 días)
   - Visualizaciones en respuestas (1 día)

   **Semana 6: Exportación y Pulido**
   - Sistema de exportación (2 días)
   - Optimizaciones de rendimiento (1 día)
   - Pruebas finales y correcciones (2 días)

4. **Consideraciones de Escalabilidad**:
   - Implementar caché agresiva para documentos procesados
   - Utilizar virtualización para documentos extensos
   - Diseñar para soporte multilingüe desde el principio
   - Habilitar lazy loading de visualizaciones

5. **KPIs de Implementación**:
   - Tiempo de carga inicial < 1.5 segundos
   - Tiempo de respuesta de UI < 100ms
   - Memoria consumida en cliente < 150MB
   - Coverage de tests > 80%
   - Accesibilidad WCAG AA en todas las pantallas

Esta propuesta completa de diseño UX/UI proporciona todos los detalles técnicos, especificaciones visuales y componentes necesarios para implementar una interfaz que aproveche al máximo la arquitectura validada de FinDocs AI, con atención especial a la experiencia de usuario, rendimiento y escalabilidad.