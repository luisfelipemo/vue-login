<script setup>
import { computed, onMounted, ref } from 'vue'

const email = ref('')
const password = ref('')
const remember = ref(true)
const showPassword = ref(false)
const submitted = ref(false)
const loading = ref(false)
const formMessage = ref('')
const apiUrl = (import.meta.env.VITE_API_URL || '/api').replace(/\/$/, '')
const isRegister = ref(false)
const confirmPassword = ref('')
const authUser = ref(null)
const restoringSession = ref(true)
const sessionVerified = ref(false)
const emailError = computed(() => email.value && !/^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(email.value))
const passwordError = computed(() => submitted.value && password.value.length > 0 && password.value.length < 8)
const confirmPasswordError = computed(() => submitted.value && isRegister.value && confirmPassword.value.length > 0 && confirmPassword.value !== password.value)

function clearSession() {
  localStorage.removeItem('authToken')
  localStorage.removeItem('authUser')
  sessionStorage.removeItem('authToken')
  sessionStorage.removeItem('authUser')
}

async function restoreSession() {
  const storage = localStorage.getItem('authToken') ? localStorage : sessionStorage
  const token = storage.getItem('authToken')
  if (!token) {
    restoringSession.value = false
    return
  }

  try {
    const response = await fetch(`${apiUrl}/auth/me`, {
      headers: { Authorization: `Bearer ${token}` },
    })
    if (!response.ok) {
      clearSession()
      return
    }
    const result = await response.json()
    authUser.value = result.user
    sessionVerified.value = true
  } catch {
    const savedUser = storage.getItem('authUser')
    if (savedUser) {
      try { authUser.value = JSON.parse(savedUser) } catch { clearSession() }
    }
  } finally {
    restoringSession.value = false
  }
}

function logout() {
  clearSession()
  authUser.value = null
  sessionVerified.value = false
  formMessage.value = ''
  password.value = ''
}

onMounted(restoreSession)

function toggleMode() {
  isRegister.value = !isRegister.value
  submitted.value = false
  formMessage.value = ''
  confirmPassword.value = ''
}

async function submitForm() {
  submitted.value = true
  formMessage.value = ''
  if (!email.value || emailError.value || !password.value || password.value.length < 8) return
  if (isRegister.value && confirmPassword.value !== password.value) return

  loading.value = true
  try {
    const endpoint = isRegister.value ? 'register' : 'login'
    const response = await fetch(`${apiUrl}/auth/${endpoint}`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ email: email.value, password: password.value }),
    })
    const result = await response.json()
    if (!response.ok) throw new Error(result.message || 'No se pudo iniciar sesión.')

    clearSession()
    const storage = remember.value ? localStorage : sessionStorage
    storage.setItem('authToken', result.token)
    storage.setItem('authUser', JSON.stringify(result.user))
    authUser.value = result.user
    sessionVerified.value = true
    formMessage.value = isRegister.value
      ? `¡Tu cuenta está lista, ${result.user.email}!`
      : `¡Bienvenido, ${result.user.email}!`
  } catch (error) {
    formMessage.value = error instanceof TypeError
      ? 'No pudimos conectar con el servidor. Comprueba que el backend esté iniciado.'
      : error.message
  } finally {
    loading.value = false
  }
}
</script>

<template>
  <main v-if="restoringSession" class="menu-shell restoring-session" aria-live="polite">
    <span class="brand"><span class="brand-mark" aria-hidden="true"><span></span><span></span><span></span></span>nube</span>
    <p>Comprobando tu sesión…</p>
  </main>

  <main v-else-if="authUser" class="menu-shell">
    <header class="menu-header">
      <div class="brand"><span class="brand-mark" aria-hidden="true"><span></span><span></span><span></span></span><span>nube</span></div>
      <div class="user-menu">
        <span class="user-status"><span></span> {{ sessionVerified ? 'Sesión activa' : 'Sesión guardada' }}</span>
        <button class="logout-button" type="button" @click="logout">Cerrar sesión</button>
      </div>
    </header>
    <section class="menu-welcome" aria-labelledby="menu-title">
      <p class="eyebrow">TU ESPACIO, A TU MANERA</p>
      <h1 id="menu-title">¡Hola de nuevo<span>.</span></h1>
      <p>Has iniciado sesión como <strong>{{ authUser.email }}</strong>.</p>
    </section>
    <section class="menu-grid" aria-label="Menú principal">
      <article class="menu-tile">
        <span class="tile-icon" aria-hidden="true">◎</span>
        <div><p class="tile-kicker">CUENTA</p><h2>Mi perfil</h2><p>{{ authUser.email }}</p></div>
      </article>
      <article class="menu-tile session-tile">
        <span class="tile-icon" aria-hidden="true">✓</span>
        <div><p class="tile-kicker">ESTADO</p><h2>{{ sessionVerified ? 'Todo listo' : 'Sin conexión' }}</h2><p>{{ sessionVerified ? 'Tu sesión está activa y verificada.' : 'No pudimos comprobar el token con el servidor.' }}</p></div>
      </article>
    </section>
    <footer class="menu-footer"><span>nube</span><span>Tu espacio personal</span></footer>
  </main>

  <main v-else class="page-shell">
    <section class="login-card" aria-labelledby="welcome-title">
      <div class="brand" aria-label="Nube">
        <span class="brand-mark" aria-hidden="true"><span></span><span></span><span></span></span>
        <span>nube</span>
      </div>
      <div class="intro">
        <p class="eyebrow">TU ESPACIO, A TU MANERA</p>
        <h1 id="welcome-title">{{ isRegister ? 'Crea tu cuenta' : 'Qué bueno verte de nuevo' }}<span>.</span></h1>
        <p class="subtitle">{{ isRegister ? 'Regístrate para empezar.' : 'Inicia sesión para continuar donde lo dejaste.' }}</p>
      </div>
      <form class="login-form" @submit.prevent="submitForm" novalidate>
        <label for="email">Correo electrónico</label>
        <div class="input-wrap" :class="{ invalid: emailError || (submitted && !email) }">
          <svg viewBox="0 0 24 24" aria-hidden="true"><rect x="3.5" y="5.5" width="17" height="13" rx="2.5"/><path d="m4.5 7 7.5 5.5L19.5 7"/></svg>
          <input id="email" v-model.trim="email" type="email" autocomplete="email" placeholder="tu@correo.com" required />
        </div>
        <p v-if="emailError" class="field-error">Escribe un correo válido.</p>
        <p v-else-if="submitted && !email" class="field-error">Este campo es obligatorio.</p>
        <div class="password-label">
          <label for="password">Contraseña</label>
          <a v-if="!isRegister" href="#recuperar" @click.prevent="formMessage = 'Conecta aquí el flujo para recuperar tu contraseña.'">¿La olvidaste?</a>
        </div>
        <div class="input-wrap" :class="{ invalid: passwordError || (submitted && !password) }">
          <svg viewBox="0 0 24 24" aria-hidden="true"><rect x="4.5" y="10" width="15" height="10" rx="2"/><path d="M8 10V7a4 4 0 0 1 8 0v3m-4 4v2"/></svg>
          <input id="password" v-model="password" :type="showPassword ? 'text' : 'password'" :autocomplete="isRegister ? 'new-password' : 'current-password'" placeholder="Mínimo 8 caracteres" required minlength="8" />
          <button class="visibility" type="button" :aria-label="showPassword ? 'Ocultar contraseña' : 'Mostrar contraseña'" @click="showPassword = !showPassword">
            <svg v-if="!showPassword" viewBox="0 0 24 24" aria-hidden="true"><path d="M2.5 12s3.4-6 9.5-6 9.5 6 9.5 6-3.4 6-9.5 6-9.5-6-9.5-6Z"/><circle cx="12" cy="12" r="2.5"/></svg>
            <svg v-else viewBox="0 0 24 24" aria-hidden="true"><path d="m3 3 18 18M10.6 6.2A10 10 0 0 1 12 6c6.1 0 9.5 6 9.5 6a15 15 0 0 1-3 3.5M6.2 6.3C3.8 7.8 2.5 12 2.5 12s3.4 6 9.5 6c1 0 2-.2 2.8-.5"/><path d="M9.9 9.9a3 3 0 0 0 4.2 4.2"/></svg>
          </button>
        </div>
        <p v-if="passwordError" class="field-error">Usa al menos 8 caracteres.</p>
        <p v-else-if="submitted && !password" class="field-error">Este campo es obligatorio.</p>
        <template v-if="isRegister">
          <label for="confirm-password" class="confirm-label">Confirmar contraseña</label>
          <div class="input-wrap" :class="{ invalid: confirmPasswordError || (submitted && !confirmPassword) }">
            <svg viewBox="0 0 24 24" aria-hidden="true"><rect x="4.5" y="10" width="15" height="10" rx="2"/><path d="M8 10V7a4 4 0 0 1 8 0v3m-4 4v2"/></svg>
            <input id="confirm-password" v-model="confirmPassword" :type="showPassword ? 'text' : 'password'" autocomplete="new-password" placeholder="Repite tu contraseña" required />
          </div>
          <p v-if="submitted && !confirmPassword" class="field-error">Confirma tu contraseña.</p>
          <p v-else-if="confirmPasswordError" class="field-error">Las contraseñas no coinciden.</p>
        </template>
        <label v-else class="remember"><input v-model="remember" type="checkbox" /><span class="checkmark" aria-hidden="true"></span><span>Recordarme</span></label>
        <button class="submit-button" type="submit" :disabled="loading">{{ loading ? 'Conectando…' : isRegister ? 'Crear cuenta' : 'Iniciar sesión' }} <span aria-hidden="true">→</span></button>
        <p v-if="formMessage" class="form-message" role="status">{{ formMessage }}</p>
      </form>
      <p class="signup">{{ isRegister ? '¿Ya tienes cuenta?' : '¿Aún no tienes cuenta?' }} <a href="#registro" @click.prevent="toggleMode">{{ isRegister ? 'Iniciar sesión' : 'Crear cuenta' }}</a></p>
      <div class="secure-note"><span aria-hidden="true">✳</span> Tu información está protegida y segura</div>
    </section>
    <aside class="visual-panel" aria-label="Ilustración abstracta">
      <div class="orb orb-one"></div><div class="orb orb-two"></div><div class="orb orb-three"></div>
      <div class="panel-copy"><span class="sparkle">✳</span><p>Un solo lugar.<br /><strong>Todo lo que importa.</strong></p><span class="panel-caption">HECHO PARA TU DÍA A DÍA</span></div>
      <span class="panel-index">01 — 03</span>
    </aside>
  </main>
</template>
