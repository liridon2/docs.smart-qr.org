# Toast Customization Options

## Aktuelle Implementierung

```tsx
toast.success('Nachricht', {
  duration: 6000,
  icon: '✅',
  style: {
    background: '#10b981',
    color: '#fff',
    fontSize: '16px',
    padding: '16px',
  }
})
```

## Weitere Anpassungsmöglichkeiten

### 1. Position ändern
In `App.tsx` den Toaster konfigurieren:

```tsx
<Toaster
  position="top-center"  // oder: bottom-center, top-left, top-right, etc.
  reverseOrder={false}
/>
```

### 2. Success Toast mit Animation

```tsx
toast.success('Zahlung erfolgreich!', {
  duration: 6000,
  icon: '🎉',
  style: {
    background: 'linear-gradient(135deg, #667eea 0%, #764ba2 100%)',
    color: '#fff',
    fontSize: '16px',
    padding: '16px',
    borderRadius: '12px',
    boxShadow: '0 10px 40px rgba(0,0,0,0.2)',
  }
})
```

### 3. Custom Toast mit mehr Inhalt

```tsx
toast.custom((t) => (
  <div
    className={`${
      t.visible ? 'animate-enter' : 'animate-leave'
    } max-w-md w-full bg-white shadow-lg rounded-lg pointer-events-auto flex ring-1 ring-black ring-opacity-5`}
  >
    <div className="flex-1 w-0 p-4">
      <div className="flex items-start">
        <div className="flex-shrink-0 pt-0.5">
          <span className="text-3xl">✅</span>
        </div>
        <div className="ml-3 flex-1">
          <p className="text-sm font-medium text-gray-900">
            Zahlung erfolgreich!
          </p>
          <p className="mt-1 text-sm text-gray-500">
            Vielen Dank für Ihren Besuch. Sie können den Tisch jetzt verlassen.
          </p>
        </div>
      </div>
    </div>
    <div className="flex border-l border-gray-200">
      <button
        onClick={() => toast.dismiss(t.id)}
        className="w-full border border-transparent rounded-none rounded-r-lg p-4 flex items-center justify-center text-sm font-medium text-indigo-600 hover:text-indigo-500"
      >
        OK
      </button>
    </div>
  </div>
))
```

### 4. Toast mit Fortschrittsbalken

```tsx
const toastId = toast.success('Zahlung erfolgreich!', {
  duration: 5000,
  style: {
    background: '#10b981',
    color: '#fff',
  }
})

// Optional: Progress bar hinzufügen
toast.loading('Verarbeite...', { id: toastId })
```

### 5. Verschiedene Farben für verschiedene Situationen

```tsx
// Erfolg (Grün)
toast.success('Zahlung erfolgreich!', {
  style: { background: '#10b981', color: '#fff' }
})

// Info (Blau)
toast('Bitte warten...', {
  icon: 'ℹ️',
  style: { background: '#3b82f6', color: '#fff' }
})

// Warnung (Orange)
toast('Achtung: Verbindung langsam', {
  icon: '⚠️',
  style: { background: '#f59e0b', color: '#fff' }
})

// Fehler (Rot)
toast.error('Zahlung fehlgeschlagen', {
  style: { background: '#ef4444', color: '#fff' }
})
```

### 6. Restaurant-themed Toast

```tsx
toast.success('Guten Appetit bezahlt!', {
  icon: '🍽️',
  duration: 5000,
  style: {
    background: 'linear-gradient(135deg, #667eea 0%, #764ba2 100%)',
    color: '#fff',
    fontSize: '17px',
    fontWeight: '600',
    padding: '20px',
    borderRadius: '16px',
    boxShadow: '0 20px 50px rgba(102, 126, 234, 0.4)',
  }
})
```

### 7. Toast mit Sound (optional)

```tsx
const playSuccessSound = () => {
  const audio = new Audio('/sounds/success.mp3')
  audio.play().catch(console.error)
}

toast.success('Zahlung erfolgreich!', {
  duration: 6000,
  icon: '✅',
  style: { background: '#10b981', color: '#fff' }
})
playSuccessSound()
```

## Empfohlene Konfiguration für SmartQR

Für die beste UX bei Restaurant-Zahlungen:

```tsx
// Erfolg
toast.success('Zahlung erfolgreich! Vielen Dank für Ihren Besuch.', {
  duration: 6000,
  icon: '✅',
  style: {
    background: '#10b981',
    color: '#fff',
    fontSize: '16px',
    padding: '16px',
    borderRadius: '12px',
    fontWeight: '500',
  },
  position: 'top-center',
})

// Fehler
toast.error('Zahlung fehlgeschlagen. Bitte versuchen Sie es erneut.', {
  duration: 5000,
  icon: '❌',
  style: {
    fontSize: '16px',
  }
})
```
