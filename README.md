-- LocalScript (Roblox) - exemplo simples sem RemoteEvent (funciona para teleporte local)
-- Pressione K para salvar posição atual, L para voltar
-- Coloque em StarterPlayerScripts

local UserInputService = game:GetService("UserInputService")
local player = game.Players.LocalPlayer

local saved = {}

local function saveCurrent(tag)
    local char = player.Character
    if not char or not char.PrimaryPart then return end
    local cf = char.PrimaryPart.CFrame
    saved[tag] = cf
    print("Posição '"..tag.."' salva.")
end

local function goto(tag)
    local char = player.Character
    if not char or not char.PrimaryPart then return end
    local cf = saved[tag]
    if not cf then
        warn("Tag não encontrada:", tag)
        return
    end
    -- Aplicar CFrame diretamente (local)
    char:SetPrimaryPartCFrame(cf)
    print("Teletransportado para '"..tag.."'.")
end

UserInputService.InputBegan:Connect(function(input, gpe)
    if gpe then return end
    if input.KeyCode == Enum.KeyCode.K then
        saveCurrent("home")
    elseif input.KeyCode == Enum.KeyCode.L then
        goto("home")
    end
end)
