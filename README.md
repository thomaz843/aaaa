-- LocalScript: SaveAndReturn.local.lua
-- Coloque em StarterPlayerScripts
-- Teclas: Z = salvar posição, X = voltar para a posição salva

local UserInputService = game:GetService("UserInputService")
local Players = game:GetService("Players")
local ReplicatedStorage = game:GetService("ReplicatedStorage")

local player = Players.LocalPlayer
local savedCFrame = nil

-- Se você usar a versão com DataStore, crie um RemoteEvent chamado "SavePosition" em ReplicatedStorage
local SavePositionRemote = ReplicatedStorage:FindFirstChild("SavePosition")

local function getHumanoidRootPart()
    local char = player.Character
    if not char then return nil end
    return char:FindFirstChild("HumanoidRootPart") or char:FindFirstChild("Torso")
end

local function notify(title, text)
    -- tenta notificar via SetCore (funciona no cliente)
    pcall(function()
        game:GetService("StarterGui"):SetCore("SendNotification", {
            Title = title;
            Text = text;
            Duration = 2;
        })
    end)
end

local function savePosition()
    local hrp = getHumanoidRootPart()
    if not hrp then
        notify("Salvar posição", "Personagem não encontrado.")
        return
    end
    savedCFrame = hrp.CFrame
    notify("Salvar posição", "Posição salva localmente.")
    -- Se existir RemoteEvent, envie ao servidor para persistir (opcional)
    if SavePositionRemote then
        -- envia só a posição (CFrame é aceito pelo RemoteEvent)
        pcall(function()
            SavePositionRemote:FireServer(savedCFrame)
        end)
    end
end

local function returnToSaved()
    local hrp = getHumanoidRootPart()
    if not hrp then
        notify("Voltar", "Personagem não encontrado.")
        return
    end
    if not savedCFrame then
        notify("Voltar", "Nenhuma posição salva.")
        return
    end
    -- Teleporta o jogador de volta à posição salva
    -- Ajuste direto do CFrame (pode causar colisões se necessário, você pode usar TweenService para suavizar)
    hrp.CFrame = savedCFrame
    notify("Voltar", "Teletransportado para a posição salva.")
end

-- Reconecta quando o personagem reaparece (salva permanece na sessão do cliente)
player.CharacterAdded:Connect(function()
    -- opcional: se quiser que ao reaparecer volte automaticamente, descomente a linha abaixo
    -- if savedCFrame then player.Character:WaitForChild("HumanoidRootPart").CFrame = savedCFrame end
end)

UserInputService.InputBegan:Connect(function(input, gameProcessed)
    if gameProcessed then return end
    if input.UserInputType == Enum.UserInputType.Keyboard then
        if input.KeyCode == Enum.KeyCode.Z then
            savePosition()
        elseif input.KeyCode == Enum.KeyCode.X then
            returnToSaved()
        end
    end
end)
