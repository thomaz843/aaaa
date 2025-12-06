-- coords_multienv.lua
-- Script multi-ambiente: Roblox (Luau), FiveM (GTA V) e Lua genérico.
-- Ele detecta o ambiente e roda apenas o trecho correspondente.
-- Em ambientes genéricos substitua getPlayerPosition() pela API do seu jogo.
-- Feito para minimizar erros usando pcall e checagens seguras.

-- UTILITÁRIOS
local function safe_get_global(name)
  local ok, val = pcall(function() return _G[name] end)
  if ok then return val end
  return nil
end

local function is_roblox()
  local g = safe_get_global("game")
  if not g then return false end
  local ok, has = pcall(function() return g.GetService ~= nil end)
  return ok and type(g.GetService) == "function"
end

local function is_fivem()
  local citizen = safe_get_global("Citizen")
  if citizen and type(citizen.CreateThread) == "function" then return true end
  if type(safe_get_global("PlayerPedId")) == "function" and type(safe_get_global("GetEntityCoords")) == "function" then
    return true
  end
  return false
end

-- Portable sleep (seconds). Tenta luasocket, então comandos do SO.
local function sleep_seconds(s)
  if s <= 0 then return end
  local socket = safe_get_global("socket")
  if socket and type(socket.sleep) == "function" then
    socket.sleep(s); return
  end
  local ok, pc = pcall(function() return package and package.config end)
  if ok and type(pc) == "string" then
    local is_unix = pc:sub(1,1) == "/"
    if is_unix then
      os.execute("sleep " .. tonumber(s))
      return
    else
      -- Windows fallback (timeout accepts integers)
      os.execute("timeout /T " .. math.max(1, math.floor(s)) .. " >NUL")
      return
    end
  end
  -- Last resort busy-wait (not ideal but safe)
  local t0 = os.time()
  while os.time() - t0 < s do end
end

-- ====== ROBLOX BRANCH ======
local function run_roblox()
  -- Executar somente em LocalPlayer context (LocalScript)
  local ok, Players = pcall(function() return game:GetService("Players") end)
  if not ok or not Players then return end
  local RunService = game:GetService("RunService")
  local player = Players.LocalPlayer
  if not player then return end

  -- Evita criar GUI duplicado
  local existing = player:FindFirstChild("PlayerGui") and player.PlayerGui:FindFirstChild("CoordsGui")
  if existing then return end

  -- Cria GUI simples
  local screenGui = Instance.new("ScreenGui")
  screenGui.Name = "CoordsGui"
  screenGui.ResetOnSpawn = false

  local frame = Instance.new("Frame", screenGui)
  frame.Size = UDim2.new(0, 260, 0, 60)
  frame.Position = UDim2.new(0, 10, 0, 10)
  frame.BackgroundTransparency = 0.4
  frame.BackgroundColor3 = Color3.fromRGB(0, 0, 0)
  frame.BorderSizePixel = 0

  local label = Instance.new("TextLabel", frame)
  label.Size = UDim2.new(1, -10, 1, -10)
  label.Position = UDim2.new(0, 5, 0, 5)
  label.BackgroundTransparency = 1
  label.TextColor3 = Color3.fromRGB(255, 255, 255)
  label.TextScaled = false
  label.TextSize = 18
  label.Font = Enum.Font.SourceSans
  label.Text = "Carregando coordenadas..."

  screenGui.Parent = player:WaitForChild("PlayerGui")

  -- Atualiza a cada frame para maior precisão
  local conn
  conn = RunService.RenderStepped:Connect(function()
    local character = player.Character
    if character then
      local hrp = character:FindFirstChild("HumanoidRootPart") or character:FindFirstChild("Humanoid") and character.Humanoid.RootPart
      if hrp and hrp.Position then
        local pos = hrp.Position
        label.Text = string.format("X: %.2f   Y: %.2f   Z: %.2f", pos.X, pos.Y, pos.Z)
      else
        label.Text = "Personagem sem HumanoidRootPart"
      end
    else
      label.Text = "Sem personagem"
    end
  end)

  -- Limpeza ao finalizar (opcional): desconectar quando o jogador sair
  player.AncestryChanged:Connect(function()
    if not player:IsDescendantOf(game) and conn then
      conn:Disconnect()
    end
  end)
end

-- ====== FIVEM BRANCH ======
local function run_fivem()
  local Citizen = safe_get_global("Citizen")
  if not Citizen or type(Citizen.CreateThread) ~= "function" then
    -- Tentativa alternativa: se funções nativas existirem, ainda assim podemos usar loop simples
    if type(safe_get_global("PlayerPedId")) ~= "function" then return end
  end

  local create_thread = function(f)
    if Citizen and type(Citizen.CreateThread) == "function" then
      Citizen.CreateThread(f)
    else
      -- Fallback: try spawn if available
      local spawn = safe_get_global("spawn")
      if type(spawn) == "function" then
        spawn(f)
      else
        -- Last resort: run function once (no looping thread available)
        -- We'll call it directly but it must manage its own loop
        pcall(f)
      end
    end
  end

  create_thread(function()
    while true do
      local ok, ped = pcall(function() return PlayerPedId() end)
      if ok and ped and ped ~= 0 then
        local ok2, coords = pcall(function() return GetEntityCoords(ped) end)
        if ok2 and coords then
          local x = coords.x or coords[1]
          local y = coords.y or coords[2]
          local z = coords.z or coords[3]
          local text = string.format("Coordenadas: X: %.3f  Y: %.3f  Z: %.3f", tonumber(x) or 0, tonumber(y) or 0, tonumber(z) or 0)
          -- Print no console F8
          pcall(print, text)
          -- Notificação segura (pode falhar se natives não existirem)
          pcall(function()
            SetNotificationTextEntry("STRING")
            AddTextComponentString(text)
            DrawNotification(false, false)
          end)
        end
      end
      -- Usa Citizen.Wait se disponível, senão fallback para sleep_seconds
      if Citizen and type(Citizen.Wait) == "function" then
        Citizen.Wait(3000)
      else
        sleep_seconds(3)
      end
    end
  end)
end

-- ====== GENERIC LUA BRANCH ======
local function run_generic()
  local log_file = "coords.txt"
  local interval_s = 1 -- segundos

  -- Placeholder: substitua esta função pela API do seu jogo que retorna x,y,z
  local function getPlayerPosition()
    -- Exemplo: retorna 0,0,0. Troque por: return player:getPosition() ou sua API.
    return 0.0, 0.0, 0.0
  end

  local function appendCoordsToFile(x, y, z)
    local f, err = io.open(log_file, "a")
    if not f then
      pcall(print, "Erro ao abrir arquivo:", err)
      return
    end
    local line = string.format("%s\tX:%.3f\tY:%.3f\tZ:%.3f\n", os.date("%Y-%m-%d %H:%M:%S"), tonumber(x) or 0, tonumber(y) or 0, tonumber(z) or 0)
    f:write(line)
    f:close()
  end

  while true do
    local ok, x, y, z = pcall(function() return getPlayerPosition() end)
    if ok then
      -- getPlayerPosition pode retornar 3 valores ou uma tabela; trata ambos
      if type(x) == "table" and x.x and x.y and x.z then
        appendCoordsToFile(x.x, x.y, x.z)
        pcall(print, string.format("Posição: X=%.3f Y=%.3f Z=%.3f", x.x, x.y, x.z))
      else
        -- se x,y,z forem valores separados
        appendCoordsToFile(x or 0, y or 0, z or 0)
        pcall(print, string.format("Posição: X=%.3f Y=%.3f Z=%.3f", x or 0, y or 0, z or 0))
      end
    else
      pcall(print, "Erro ao obter posição do jogador")
    end
    sleep_seconds(interval_s)
  end
end

-- ====== BOOTSTRAP: detecta e executa a branch apropriada ======
local function main()
  if is_roblox() then
    local ok, err = pcall(run_roblox)
    if not ok then pcall(print, "Erro ao iniciar branch Roblox:", err) end
    return
  end

  if is_fivem() then
    local ok, err = pcall(run_fivem)
    if not ok then pcall(print, "Erro ao iniciar branch FiveM:", err) end
    return
  end

  -- Fallback genérico
  local ok, err = pcall(run_generic)
  if not ok then pcall(print, "Erro ao iniciar branch genérica:", err) end
end

-- Inicia
main()
