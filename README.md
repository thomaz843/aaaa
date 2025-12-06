-- ServerScript: SavePositionServer.server.lua
-- Coloque em ServerScriptService
-- Cria/usa um RemoteEvent chamado "SavePosition" dentro de ReplicatedStorage
-- Armazena apenas a posição (Vector3) no DataStore. Teste com API Services habilitado.

local DataStoreService = game:GetService("DataStoreService")
local Players = game:GetService("Players")
local ReplicatedStorage = game:GetService("ReplicatedStorage")

local savedStore = DataStoreService:GetDataStore("PlayerSavedPositions_v1")

-- Certifique-se de que o RemoteEvent exista
local saveEvent = ReplicatedStorage:FindFirstChild("SavePosition")
if not saveEvent then
    saveEvent = Instance.new("RemoteEvent")
    saveEvent.Name = "SavePosition"
    saveEvent.Parent = ReplicatedStorage
end

-- Recebe do cliente uma CFrame e salva a posição no DataStore
saveEvent.OnServerEvent:Connect(function(player, cf)
    -- Validação básica
    if typeof(cf) ~= "CFrame" then return end
    local pos = { x = cf.Position.X, y = cf.Position.Y, z = cf.Position.Z }
    local key = tostring(player.UserId)
    local success, err = pcall(function()
        savedStore:SetAsync(key, pos)
    end)
    if not success then
        warn("Erro ao salvar posição para "..player.Name..": "..tostring(err))
    end
end)

-- Ao entrar, tenta ler a posição salva e aplica quando o personagem aparece
Players.PlayerAdded:Connect(function(player)
    local key = tostring(player.UserId)
    local success, pos = pcall(function()
        return savedStore:GetAsync(key)
    end)
    if success and pos then
        player.CharacterAdded:Connect(function(char)
            local hrp = char:WaitForChild("HumanoidRootPart", 5)
            if hrp then
                -- aplica posição salva
                local cf = CFrame.new(pos.x, pos.y, pos.z)
                hrp.CFrame = cf
            end
        end)
    end
end)
