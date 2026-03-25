# 🔌 Guia de Integração — Novos Componentes

## O que foi criado

| Arquivo | Descrição |
|---|---|
| `src/components/dashboard/viral-score-panel.tsx` | Score viral 0-100 + gancho adaptado por post |
| `src/components/dashboard/content-generator-modal.tsx` | Gerador de carrossel + roteiro de Reel |
| `src/components/dashboard/pipeline-progress-bar.tsx` | Barra de progresso visual do pipeline |
| `supabase/migrations/20260322000000_viral_score_content_generator.sql` | Migration de banco |

---

## 1. Rodar a migration

Abra o Supabase SQL Editor e execute:
```
supabase/migrations/20260322000000_viral_score_content_generator.sql
```

---

## 2. Integrar o `ViralScorePanel` na tabela de reels

Em `data-table.tsx`, dentro do render de cada linha de reel de competidor:

```tsx
import { ViralScorePanel, type ViralAnalysis } from "./viral-score-panel"
import { ContentGeneratorModal } from "./content-generator-modal"

// State no componente pai (Index.tsx ou data-table.tsx):
const [generatorOpen, setGeneratorOpen] = useState(false)
const [activeAnalysis, setActiveAnalysis] = useState<ViralAnalysis | null>(null)

// Dentro do render de cada reel:
<ViralScorePanel
  reelId={reel.id}
  caption={reel.caption}
  views={reel.views}
  likes={reel.likes}
  comments={reel.comments}
  contentType={reel.contentType}
  ownerUsername={reel.user}
  profileName="@djairrotta"
  profileNiche="Advogado especialista em direito bancário. Tom educativo e acessível."
  onAnalysisDone={async (analysis) => {
    // Persistir no banco
    await supabase.rpc("save_viral_analysis", {
      p_reel_id: reel.id,
      p_table: "reels_competidores",
      p_score: analysis.viralScore,
      p_hook_type: analysis.hookType,
      p_adapted_hook: analysis.adaptedHook,
      p_reach: analysis.estimatedReach,
      p_format: analysis.bestFormat,
      p_analysis: analysis,
    })
  }}
  onGenerateContent={(analysis) => {
    setActiveAnalysis(analysis)
    setGeneratorOpen(true)
  }}
/>
```

---

## 3. Abrir o `ContentGeneratorModal`

```tsx
<ContentGeneratorModal
  open={generatorOpen}
  onOpenChange={setGeneratorOpen}
  analysis={activeAnalysis}
  sourceCaption={selectedReel?.caption}
  ownerUsername={selectedReel?.user}
  profileName="@djairrotta"
  profileNiche="Advogado especialista em direito bancário, juro abusivo, defesa do consumidor."
  profileHashtags="#advogado #direitobancario #juroabusivo #bancario #consumidor"
  onAddToQueue={async (type, content) => {
    const { data: { user } } = await supabase.auth.getUser()
    
    const items = []
    if (type === "carrossel" || type === "ambos") {
      items.push({
        user_id: user?.id,
        content_type: "carousel",
        caption: content.carousel.caption,
        carousel_slides: content.carousel.slides,
        viral_score: activeAnalysis?.viralScore || 0,
        status: "pending_approval",
      })
    }
    if (type === "reel" || type === "ambos") {
      items.push({
        user_id: user?.id,
        content_type: "reel",
        caption: content.reel.caption,
        reel_script: content.reel.script,
        viral_score: activeAnalysis?.viralScore || 0,
        status: "pending_approval",
      })
    }
    
    await supabase.from("publish_queue").insert(items)
    toast.success("Adicionado à fila de aprovação!")
    setPendingQueueCount(c => c + items.length)
  }}
/>
```

---

## 4. Integrar o `PipelineProgressBar` no modal de logs

Em `pipeline-logs-modal.tsx`:

```tsx
import { PipelineProgressBar, PipelineLogTerminal, usePipelineStepsFromLogs } from "./pipeline-progress-bar"

// Dentro do componente:
const steps = usePipelineStepsFromLogs(logs, isRunning)

// No JSX, antes dos logs:
<PipelineProgressBar steps={steps} isRunning={isRunning} className="mb-4" />
<PipelineLogTerminal logs={logs} isRunning={isRunning} />
```

**Versão compacta no topbar do Dashboard (Index.tsx):**

```tsx
import { PipelineProgressBar, usePipelineStepsFromLogs } from "@/components/dashboard/pipeline-progress-bar"

// Dentro do header, ao lado do botão "Rodar Pipeline":
{isPipelineRunning && (
  <PipelineProgressBar
    steps={usePipelineStepsFromLogs(pipelineLogs, isPipelineRunning)}
    isRunning={isPipelineRunning}
    compact
    className="min-w-[200px]"
  />
)}
```

---

## 5. Exibir score viral na sidebar de banco de dados

Na `database-modal.tsx`, ao listar `reels_competidores`, adicione o badge de score:

```tsx
{reel.viralScore > 0 && (
  <span className={cn(
    "text-xs font-mono font-bold px-1.5 py-0.5 rounded",
    reel.viralScore >= 75 ? "bg-green-500/20 text-green-400" :
    reel.viralScore >= 50 ? "bg-yellow-500/20 text-yellow-400" :
    "bg-muted/30 text-muted-foreground"
  )}>
    {reel.viralScore}
  </span>
)}
```

---

## 6. O que falta no backend (gap analysis completo)

### ❌ Edge functions ausentes / incompletas

| Função | Status | Problema |
|---|---|---|
| `publish-instagram` | ⚠️ Parcial | Falta suporte a carrossel (múltiplas mídias via container) |
| `analyze-content` | ⚠️ Parcial | Não salva `viral_score` no banco após análise |
| `run-pipeline` | ⚠️ Parcial | Não inclui etapa de score viral automático por reel |
| `viral-score-batch` | ❌ Não existe | Precisa criar: roda score nos top N reels após scraping |
| `save-carousel-content` | ❌ Não existe | Precisa criar: salva slides em `generated_content` |

### ❌ Tabelas/campos que faltavam (já adicionados na migration)

- `reels_competidores.viral_score` → adicionado
- `reels_competidores.adapted_hook` → adicionado
- `generated_content.carousel_slides` → adicionado
- `generated_content.reel_script` → adicionado
- `publish_queue.carousel_slides` → adicionado
- `publish_queue.viral_score` → adicionado

### ✅ O que já funciona e pode ser reaproveitado

- `generate-content` → já suporta `openrouter`, `anthropic`, `groq` — os novos componentes chamam ele diretamente
- `publish_queue` → já existe, agora com campos extras para carrossel/reel
- `poll-instagram-dms` → funciona perfeitamente, não precisa alterar
- Kanban → sincroniza automaticamente com reels, continua funcionando
