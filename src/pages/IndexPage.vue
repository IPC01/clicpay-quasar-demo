<template>
  <q-page class="app-container q-pa-md">
    <header class="row items-center justify-between q-mb-md">
      <div>
        <h1 class="text-h5 q-ma-none text-weight-bold">ClicPay • Quasar Demo</h1>
        <div class="help text-caption">Formato: apenas 9 dígitos locais (ex: <code>841234567</code>)</div>
      </div>
      <div class="help text-caption">Servidor: <strong>/api/clicpay/*</strong></div>
    </header>

    <div class="row q-col-gutter-md">
      <!-- Coluna de configurações -->
      <div class="col-12 col-md-6">
        <q-card class="card q-pa-md" flat bordered>
          <q-card-section>
            <h2 class="text-h6 q-ma-none q-mb-md">1) Configurações</h2>

            <div class="choices">
              <div class="text-subtitle2 q-mb-xs">Provedor</div>
              <q-option-group :options="providerOptions" type="radio" v-model="provider" inline class="q-mb-md" />

              <q-separator class="q-my-md" />

              <div class="text-subtitle2 q-mb-xs">Tipo de Transação</div>
              <q-option-group :options="txTypeOptions" type="radio" v-model="txType" inline />
            </div>
          </q-card-section>
        </q-card>

        <q-card class="card q-pa-md q-mt-md" flat bordered>
          <q-card-section>
            <h2 class="text-h6 q-ma-none q-mb-md">2) Dados da Transação</h2>

            <q-input label="Número de telefone" type="tel" inputmode="numeric" placeholder="Ex: 841234567"
              v-model="msisdn" :error="!!error" :error-message="error" class="q-mb-md">
              <template v-slot:hint>
                M-Pesa: 84/85xxxxxxx • e-Mola: 86/87xxxxxxx
              </template>
            </q-input>

            <q-input label="Valor (MZN)" type="number" min="1" step="0.01" v-model="amount" class="q-mb-md" />

            <q-input label="Referência" type="text" maxlength="20" v-model="reference" class="q-mb-md" />

            <q-input label="Notas" type="text" maxlength="255" v-model="notes" class="q-mb-md" />

            <q-banner :class="`status-${status.type} rounded-borders q-mb-md`" v-if="status.message">
              {{ status.message }}
            </q-banner>

            <div class="actions row q-col-gutter-sm">
              <div class="col">
                <q-btn color="primary" label="Enviar" :loading="isSubmitting" :disable="!!error" @click="handleSubmit"
                  class="full-width" />
              </div>
              <div class="col">
                <q-btn color="grey" label="Limpar" @click="handleClear" class="full-width" />
              </div>
            </div>
          </q-card-section>
        </q-card>
      </div>

      <!-- Coluna de logs -->
      <div class="col-12 col-md-6">
        <q-card class="card q-pa-md" flat bordered>
          <q-card-section>
            <div class="row items-center justify-between">
              <h2 class="text-h6 q-ma-none">3) Logs</h2>
              <q-btn color="grey" label="Limpar Logs" @click="handleClearLogs" size="sm" />
            </div>

            <q-scroll-area class="log q-mt-md" style="height: 500px;">
              <div v-for="(log, index) in logs" :key="index" class="log-entry">
                <pre>{{ log }}</pre>
              </div>
            </q-scroll-area>
          </q-card-section>
        </q-card>
      </div>
    </div>
  </q-page>
</template>

<script>
import { ref } from 'vue'

export default {
  name: 'IndexPage',
  setup() {
    const provider = ref('mpesa')
    const txType = ref('c2b')
    const msisdn = ref('')
    const amount = ref('5')
    const reference = ref('ReactDemo')
    const notes = ref('Pagamento Demo React')
    const error = ref('')
    const status = ref({ message: '', type: '' })
    const isSubmitting = ref(false)
    const logs = ref(['Sistema inicializado. Aguardando operação...'])

    const providerOptions = [
      { label: 'M-Pesa', value: 'mpesa' },
      { label: 'e-Mola', value: 'emola' }
    ]

    const txTypeOptions = [
      { label: 'C2B', value: 'c2b' },
      { label: 'B2C', value: 'b2c' }
    ]

    const addLog = (message, type = 'info') => {
      const time = new Date().toLocaleTimeString()
      let formattedMessage = message
      let prefix = ''

      switch (type) {
        case 'error':
          prefix = '❌ '

          // Simplificar mensagens de erro
          if (typeof message === 'string') {
            if (message.includes("Failed to fetch")) {
              formattedMessage = JSON.stringify({ error: "Erro de conexão: Falha ao conectar com o servidor" }, null, 2)
            } else if (message.includes("timed out")) {
              formattedMessage = JSON.stringify({ error: "Tempo esgotado: A requisição demorou muito para responder" }, null, 2)
            } else {
              // Tentar extrair apenas a mensagem principal de erro
              try {
                if (message.includes('"message":')) {
                  const start = message.indexOf('"message":') + 10
                  const end = message.indexOf('",', start)
                  if (start > 0 && end > start) {
                    const errorMsg = message.substring(start, end)
                    formattedMessage = JSON.stringify({ error: errorMsg }, null, 2)
                  }
                }
              } catch {
                // Usando catch sem parâmetro
                formattedMessage = JSON.stringify({ error: "Erro desconhecido" }, null, 2)
              }
            }
          }
          break

        case 'success':
          prefix = '✅ '

          // Formatar respostas de sucesso como JSON
          try {
            if (typeof message === 'string') {
              formattedMessage = JSON.stringify(JSON.parse(message), null, 2)
            } else if (typeof message === 'object') {
              formattedMessage = JSON.stringify(message, null, 2)
            }
          } catch {
            // Manter a mensagem original se não for JSON
          }
          break

        case 'request':
          prefix = '➡️ '

          // Formatar requisições como JSON
          try {
            if (typeof message === 'string') {
              formattedMessage = JSON.stringify(JSON.parse(message), null, 2)
            } else if (typeof message === 'object') {
              formattedMessage = JSON.stringify(message, null, 2)
            }
          } catch {
            // Manter a mensagem original se não for JSON
          }
          break

        default:
          prefix = 'ℹ️ '
      }

      // Para mensagens de erro, mostrar apenas o resumo
      if (type === "error") {
        logs.value.push(`${time} ${prefix} Erro\n${formattedMessage}`)
      } else {
        logs.value.push(`${time} ${prefix}\n${formattedMessage}`)
      }
    }

    const validateForm = () => {
      if (!/^\d+$/.test(msisdn.value)) {
        error.value = 'O número deve conter apenas dígitos (0-9)'
        return false
      }
      if (msisdn.value.length !== 9) {
        error.value = 'O número deve ter exatamente 9 dígitos'
        return false
      }
      if (provider.value === 'mpesa' && !/^(84|85)\d{7}$/.test(msisdn.value)) {
        error.value = 'M-Pesa: deve começar com 84 ou 85'
        return false
      }
      if (provider.value === 'emola' && !/^(86|87)\d{7}$/.test(msisdn.value)) {
        error.value = 'e-Mola: deve começar com 86 ou 87'
        return false
      }

      error.value = ''
      return true
    }

    const handleSubmit = async () => {
      if (!validateForm()) {
        status.value = { message: 'Número inválido, verifique os dados.', type: 'error' }
        addLog('Número inválido, verifique os dados.', 'error')
        return
      }

      isSubmitting.value = true
      status.value = { message: 'Enviando requisição...', type: 'info' }

      // Use valores hardcoded para testar primeiro
      const WALLETS = {
        mpesa: '946455', // COLOQUE SEU VALOR REAL AQUI
        emola: '946454'  // COLOQUE SEU VALOR REAL AQUI
      }

      const API_TOKEN = 'eyJ0eXAiOiJKV1QiLCJhbGciOiJSUzI1NiJ9.eyJhdWQiOiIwMTk4Mzc1Yy0xZTNjLTcxOTEtYjdlMy1lZTFiMGFlNGUwNGEiLCJqdGkiOiIyODhkMWMzYTNmNTRhMjU5ZjE0MmRiZmRiZjdmYzUwNDcyMzI5Nzk2M2UzYWM0ZmM3YmYyMmU5OGM1MDdiYmFlZDBjZGMxYTE1Mzc0MzcyNSIsImlhdCI6MTc1NjQ0NzY3My43MTM1MTUsIm5iZiI6MTc1NjQ0NzY3My43MTM2MjgsImV4cCI6MjM4NzU5OTY3My43MDYxMDEsInN1YiI6IjEwIiwic2NvcGVzIjpbXX0.oNSTvlgG6id0C2M1VkMGDgdRlPUCAyzgT2ESX0x6FCBhKLFWy9ApICEz3zQfOQ6Ex_7ZoEk0YZLFt6W6J6LJGVvGwnG2VlH1PnPZvtuZhyzS8mkiKlWIDZe6dkRxM8wac62D0OZZEUrurreTSDITY2crf9aHXHpoaZkxM_Tn0JXs_oBYI57PfKBGZmbrPNuoXCmFq_sA31B_Rb34ScC46TUSSjniEyHEIjMKf9Tv1V5G9HZocWBlw1gO9FW2DOFfsZ55fkEvSW2QZ6nuQMXqNFRh7Kgx46rpsyKMsSzpYyq_09krSepSqqUZmw17AQzt81glTr4CNIlnIO89D4QCQFlgEMFeKkaUL8wjoVg8qcsHhGXI4APsoXdSQDUKr9G5jIj9fCpVNyPKd9C2srKci2F_S9GXb1XyS2TDaLR9qMXqc4MBSsgYHh70bUkbkq9CA9KB1RwRmYVb7Qn1d5lxgT8jPBEJJw9siMSu1RZLuvllwfkHeihZjgp6_PU3s_bnWW5TVNXZBqzYVK0y-RjItVKbptzOaBatC8KYgJA-i6ktijrDoL1_xrQ0931f3eoftnyoH3m9LE_0OSD8XePFfM5rz5gvML57aX1dGME2vwWcW2-zekT3SIwIiRUS1razwwPUZS7f-2vMUCk_UzgvjgayoNQCJSyb28EQ1wlpi6o' // COLOQUE SEU TOKEN REAL AQUI

      const payload = {
        msisdn: msisdn.value,
        amount: parseFloat(amount.value),
        reference_description: reference.value,
        internal_notes: notes.value
      }

      // Construir a URL da API
      const wallet = WALLETS[provider.value]
      const apiUrl = `https://clicpay.co.mz/api/v1/wallets/${wallet}/${txType.value}/${provider.value}`

      // Debug
      addLog(`Wallet: ${wallet}`, 'info')
      addLog(`Token: ${API_TOKEN ? 'Token presente' : 'Token ausente'}`, 'info')

      addLog(`Requisição POST: ${apiUrl}`, 'request')
      addLog(JSON.stringify(payload, null, 2), 'request')

      try {
        const response = await fetch(apiUrl, {
          method: 'POST',
          headers: {
            'Content-Type': 'application/json',
            'Accept': 'application/json',
            'Authorization': `Bearer ${API_TOKEN}`
          },
          body: JSON.stringify(payload)
        })

        const responseData = await response.json()

        if (!response.ok) {
          throw new Error(`Erro ${response.status}: ${JSON.stringify(responseData)}`)
        }

        addLog(JSON.stringify(responseData, null, 2), 'success')
        status.value = { message: 'Requisição enviada com sucesso!', type: 'success' }

        // Limpar campo do número após sucesso
        msisdn.value = ''
      } catch (err) {
        addLog(err.message, 'error')
        status.value = { message: 'Erro ao enviar requisição.', type: 'error' }
      } finally {
        isSubmitting.value = false
      }
    }

    const handleClear = () => {
      msisdn.value = ''
      amount.value = '5'
      reference.value = 'ReactDemo'
      notes.value = 'Pagamento Demo React'
      error.value = ''
      status.value = { message: '', type: '' }
      provider.value = 'mpesa'
      txType.value = 'c2b'
    }

    const handleClearLogs = () => {
      logs.value = ['Logs limpos. Sistema pronto para novas operações.']
    }

    return {
      provider,
      txType,
      msisdn,
      amount,
      reference,
      notes,
      error,
      status,
      isSubmitting,
      logs,
      providerOptions,
      txTypeOptions,
      handleSubmit,
      handleClear,
      handleClearLogs
    }
  }
}
</script>

<style>
:root {
  --bg1: #1e293b;
  --bg2: #334155;
  --panel: #f8fafc;
  --card-bg: #ffffff;
  --muted: #64748b;
  --text: #0f172a;
  --accent: #22c55e;
  --accent2: #10b981;
  --danger: #ef4444;
  --warn: #f59e0b;
  --link: #3b82f6;
  --border: #e2e8f0;
  --shadow: rgba(0, 0, 0, 0.1);
}

* {
  box-sizing: border-box;
  margin: 0;
  padding: 0;
}

body {
  font-family: 'Inter', system-ui, -apple-system, "Segoe UI", Roboto, Arial, sans-serif;
  color: var(--text);
  background: linear-gradient(135deg, var(--bg1) 0%, var(--bg2) 100%);
  min-height: 100vh;
  padding: 20px;
  line-height: 1.6;
}

.app-container {
  max-width: 1400px;
  margin: 0 auto;
  background: var(--panel);
  border-radius: 16px;
  box-shadow: 0 20px 40px var(--shadow);
  overflow: hidden;
}

/* Header */
.header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 24px 32px;
  background: var(--card-bg);
  border-bottom: 1px solid var(--border);
}

.header h1 {
  font-size: 24px;
  font-weight: 700;
  color: var(--text);
}

.help {
  font-size: 14px;
  color: var(--muted);
  margin-top: 4px;
}

/* Main layout */
.main-content {
  display: flex;
  min-height: calc(100vh - 200px);
}

/* Form column */
.form-column {
  flex: 1;
  padding: 32px;
  background: var(--card-bg);
  display: flex;
  flex-direction: column;
  gap: 24px;
}

/* Log column */
.log-column {
  flex: 1;
  padding: 32px;
  background: var(--panel);
  border-left: 1px solid var(--border);
  display: flex;
  flex-direction: column;
}

/* Cards */
.card {
  background: var(--card-bg);
  border-radius: 12px;
  padding: 24px;
  border: 1px solid var(--border);
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.04);
}

.card h2 {
  font-size: 18px;
  font-weight: 600;
  color: var(--text);
  margin-bottom: 16px;
}

/* Form elements */
.form-group {
  margin-bottom: 20px;
}

.form-label {
  display: block;
  font-size: 14px;
  font-weight: 500;
  color: var(--text);
  margin-bottom: 6px;
}

.form-input {
  width: 100%;
  padding: 12px 16px;
  border: 2px solid var(--border);
  border-radius: 8px;
  background: var(--card-bg);
  color: var(--text);
  font-size: 14px;
  transition: all 0.2s ease;
}

.form-input:focus {
  outline: none;
  border-color: var(--accent);
  box-shadow: 0 0 0 3px rgba(34, 197, 94, 0.1);
}

.form-input.error {
  border-color: var(--danger);
}

.input-hint {
  font-size: 12px;
  color: var(--muted);
  margin-top: 4px;
}

.error-message {
  color: var(--danger);
  font-size: 13px;
  margin-top: 6px;
  font-weight: 500;
}

/* Radio buttons */
.radio-group {
  display: flex;
  gap: 12px;
  flex-wrap: wrap;
  margin-top: 8px;
}

.radio-option {
  display: flex;
  align-items: center;
  gap: 8px;
  padding: 10px 16px;
  border: 2px solid var(--border);
  border-radius: 8px;
  cursor: pointer;
  transition: all 0.2s ease;
  background: var(--card-bg);
}

.radio-option:hover {
  border-color: var(--accent);
  background: rgba(34, 197, 94, 0.05);
}

.radio-option.selected {
  border-color: var(--accent);
  background: rgba(34, 197, 94, 0.1);
  color: var(--accent);
  font-weight: 500;
}

.radio-input {
  appearance: none;
  width: 16px;
  height: 16px;
  border: 2px solid var(--border);
  border-radius: 50%;
  position: relative;
  transition: all 0.2s ease;
}

.radio-input:checked {
  border-color: var(--accent);
  background: var(--accent);
}

.radio-input:checked::after {
  content: '';
  position: absolute;
  width: 6px;
  height: 6px;
  background: white;
  border-radius: 50%;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
}

/* Buttons */
.button-group {
  display: flex;
  gap: 12px;
  margin-top: 16px;
}

.btn {
  flex: 1;
  padding: 12px 20px;
  border: none;
  border-radius: 8px;
  font-size: 14px;
  font-weight: 600;
  cursor: pointer;
  transition: all 0.2s ease;
  display: flex;
  align-items: center;
  justify-content: center;
  gap: 8px;
}

.btn-primary {
  background: var(--accent);
  color: white;
}

.btn-primary:hover:not(:disabled) {
  background: var(--accent2);
  transform: translateY(-1px);
  box-shadow: 0 4px 12px rgba(34, 197, 94, 0.3);
}

.btn-secondary {
  background: var(--card-bg);
  color: var(--text);
  border: 2px solid var(--border);
}

.btn-secondary:hover:not(:disabled) {
  border-color: var(--muted);
  background: var(--panel);
}

.btn:disabled {
  opacity: 0.6;
  cursor: not-allowed;
  transform: none !important;
}

.btn-loading {
  position: relative;
  color: transparent;
}

.btn-loading::after {
  content: '';
  position: absolute;
  width: 16px;
  height: 16px;
  border: 2px solid transparent;
  border-top: 2px solid currentColor;
  border-radius: 50%;
  animation: spin 1s linear infinite;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
  color: white;
}

@keyframes spin {
  to {
    transform: translate(-50%, -50%) rotate(360deg);
  }
}

/* Status banner */
.status-banner {
  padding: 12px 16px;
  border-radius: 8px;
  font-size: 14px;
  font-weight: 500;
  margin: 16px 0;
  display: none;
}

.status-banner.show {
  display: block;
}

.status-banner.success {
  background: rgba(34, 197, 94, 0.1);
  color: var(--accent);
  border: 1px solid rgba(34, 197, 94, 0.2);
}

.status-banner.error {
  background: rgba(239, 68, 68, 0.1);
  color: var(--danger);
  border: 1px solid rgba(239, 68, 68, 0.2);
}

.status-banner.info {
  background: rgba(59, 130, 246, 0.1);
  color: var(--link);
  border: 1px solid rgba(59, 130, 246, 0.2);
}

/* Log section */
.log-header {
  display: flex;
  justify-content: between;
  align-items: center;
  margin-bottom: 16px;
}

.log-header h2 {
  flex: 1;
  margin: 0;
}

.log-container {
  flex: 1;
  background: var(--card-bg);
  border: 1px solid var(--border);
  border-radius: 8px;
  overflow: hidden;
  display: flex;
  flex-direction: column;
}

.log-content {
  flex: 1;
  padding: 16px;
  font-family: 'JetBrains Mono', 'Fira Code', 'Consolas', monospace;
  font-size: 13px;
  line-height: 1.5;
  overflow-y: auto;
  max-height: 600px;
  background: #fafafa;
}

.log-entry {
  margin-bottom: 12px;
  padding-bottom: 12px;
  border-bottom: 1px solid var(--border);
  word-wrap: break-word;
}

.log-entry:last-child {
  border-bottom: none;
  margin-bottom: 0;
}

.log-entry pre {
  background: var(--card-bg);
  padding: 12px;
  border-radius: 6px;
  border: 1px solid var(--border);
  overflow-x: auto;
  white-space: pre-wrap;
  color: var(--text);
  margin-top: 4px;
}

.log-time {
  font-weight: 600;
  color: var(--muted);
}

.log-info {
  color: #0369a1;
}

.log-success {
  color: #15803d;
}

.log-error {
  color: #dc2626;
}

.log-request {
  color: #7c3aed;
}

/* Separator */
.separator {
  height: 1px;
  background: var(--border);
  margin: 20px 0;
}

/* Code styling */
code {
  background: rgba(0, 0, 0, 0.05);
  padding: 2px 6px;
  border-radius: 4px;
  font-family: 'JetBrains Mono', 'Fira Code', monospace;
  font-size: 13px;
}

/* Responsive */
@media (max-width: 1024px) {
  .main-content {
    flex-direction: column;
  }

  .log-column {
    border-left: none;
    border-top: 1px solid var(--border);
  }

  .header {
    flex-direction: column;
    align-items: flex-start;
    gap: 8px;
  }
}

@media (max-width: 768px) {
  body {
    padding: 12px;
  }

  .form-column,
  .log-column {
    padding: 20px;
  }

  .header {
    padding: 20px;
  }

  .radio-group {
    flex-direction: column;
  }

  .button-group {
    flex-direction: column;
  }
}
</style>
