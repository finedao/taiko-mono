<script lang="ts">
  import type { Hash } from '@wagmi/core';
  import { onDestroy, tick } from 'svelte';
  import { t } from 'svelte-i18n';
  import { type Address, getAddress } from 'viem';

  import { routingContractsMap } from '$bridgeConfig';
  import { chainConfig } from '$chainConfig';
  import { Button } from '$components/Button';
  import { Card } from '$components/Card';
  import { ChainSelectorWrapper } from '$components/ChainSelector';
  import { successToast } from '$components/NotificationToast';
  import { infoToast } from '$components/NotificationToast/NotificationToast.svelte';
  import { OnAccount } from '$components/OnAccount';
  import { OnNetwork } from '$components/OnNetwork';
  import { Step, Stepper } from '$components/Stepper';
  import { bridges, type BridgeTransaction, MessageStatus, type NFTApproveArgs } from '$libs/bridge';
  import { hasBridge } from '$libs/bridge/bridges';
  import type { ERC721Bridge } from '$libs/bridge/ERC721Bridge';
  import type { ERC1155Bridge } from '$libs/bridge/ERC1155Bridge';
  import { fetchNFTs } from '$libs/bridge/fetchNFTs';
  import { getBridgeArgs } from '$libs/bridge/getBridgeArgs';
  import { handleBridgeError } from '$libs/bridge/handleBridgeErrors';
  import { bridgeTxService } from '$libs/storage';
  import { ETHToken, type NFT, TokenType } from '$libs/token';
  import { getCrossChainAddress } from '$libs/token/getCrossChainAddress';
  import { getTokenWithInfoFromAddress } from '$libs/token/getTokenWithInfoFromAddress';
  import { getConnectedWallet } from '$libs/util/getConnectedWallet';
  import { type Account, account } from '$stores/account';
  import { type Network, network } from '$stores/network';
  import { pendingTransactions } from '$stores/pendingTransactions';

  import Actions from './Actions.svelte';
  import type AddressInput from './AddressInput/AddressInput.svelte';
  import type Amount from './Amount.svelte';
  import type IdInput from './IDInput/IDInput.svelte';
  import ImportStep from './NFTBridgeSteps/ImportStep.svelte';
  import ReviewStep from './NFTBridgeSteps/ReviewStep.svelte';
  import type { ProcessingFee } from './ProcessingFee';
  import type Recipient from './Recipient.svelte';
  import {
    activeBridge,
    bridgeService,
    destNetwork as destinationChain,
    enteredAmount,
    processingFee,
    recipientAddress,
    selectedToken,
  } from './state';
  import { NFTSteps } from './types';

  let amountComponent: Amount;
  let recipientComponent: Recipient;
  let processingFeeComponent: ProcessingFee;
  let actionsComponent: Actions;
  let importMethod: 'scan' | 'manual' = 'scan';
  let nftIdArray: number[] = [];

  function onNetworkChange(newNetwork: Network, oldNetwork: Network) {
    updateForm();

    if (newNetwork) {
      const destChainId = $destinationChain?.id;
      if (!$destinationChain?.id) return;
      // determine if we simply swapped dest and src networks
      if (newNetwork.id === destChainId) {
        destinationChain.set(oldNetwork);
        return;
      }
      // check if the new network has a bridge to the current dest network
      if (hasBridge(newNetwork.id, $destinationChain?.id)) {
        destinationChain.set(oldNetwork);
      } else {
        // if not, set dest network to null
        $destinationChain = null;
      }
    }
  }

  const runValidations = () => {
    if (amountComponent) amountComponent.validateAmount();
    if (addressInputComponent) addressInputComponent.validateAddress();
  };

  function onAccountChange(account: Account) {
    updateForm();
    if (account && account.isDisconnected) {
      $selectedToken = null;
      $destinationChain = null;
    }
  }

  function updateForm() {
    tick().then(() => {
      if (importMethod === 'manual') {
        // run validations again if we are in manual mode
        runValidations();
      } else {
        resetForm();
      }
    });
  }

  async function approve() {
    try {
      if (!$selectedToken || !$network || !$destinationChain) return;
      const type: TokenType = $selectedToken.type;
      const walletClient = await getConnectedWallet($network.id);
      let tokenAddress = await getAddress($selectedToken.addresses[$network.id]);

      if (!tokenAddress) {
        const crossChainAddress = await getCrossChainAddress({
          token: $selectedToken,
          srcChainId: $network.id,
          destChainId: $destinationChain.id,
        });
        if (!crossChainAddress) throw new Error('cross chain address not found');
        tokenAddress = crossChainAddress;
      }
      if (!tokenAddress) {
        throw new Error('token address not found');
      }
      const tokenIds =
        nftIdArray.length > 0 ? nftIdArray.map((num) => BigInt(num)) : selectedNFT.map((nft) => BigInt(nft.tokenId));

      let txHash: Hash;

      const spenderAddress =
        type === TokenType.ERC1155
          ? routingContractsMap[$network.id][$destinationChain.id].erc1155VaultAddress
          : routingContractsMap[$network.id][$destinationChain.id].erc721VaultAddress;

      const args: NFTApproveArgs = { tokenIds: tokenIds!, tokenAddress, spenderAddress, wallet: walletClient };
      txHash = await (bridges[type] as ERC721Bridge | ERC1155Bridge).approve(args);

      const { explorer } = chainConfig[$network.id].urls;

      if (txHash)
        infoToast({
          title: $t('bridge.actions.approve.tx.title'),
          message: $t('bridge.actions.approve.tx.message', {
            values: {
              token: $selectedToken.symbol,
              url: `${explorer}/tx/${txHash}`,
            },
          }),
        });

      await pendingTransactions.add(txHash, $network.id);

      actionsComponent.checkTokensApproved();

      successToast({
        title: $t('bridge.actions.approve.success.title'),
        message: $t('bridge.actions.approve.success.message', {
          values: {
            token: $selectedToken.symbol,
          },
        }),
      });
    } catch (err) {
      console.error(err);
      handleBridgeError(err as Error);
    }
  }

  async function bridge() {
    if (!$bridgeService || !$selectedToken || !$network || !$destinationChain || !$account?.address) return;

    try {
      const walletClient = await getConnectedWallet($network.id);
      const commonArgs = {
        to: $recipientAddress || $account.address,
        wallet: walletClient,
        srcChainId: $network.id,
        destChainId: $destinationChain.id,
        fee: $processingFee,
      };

      const tokenIds =
        nftIdArray.length > 0 ? nftIdArray.map((num) => BigInt(num)) : selectedNFT.map((nft) => BigInt(nft.tokenId));

      const bridgeArgs = await getBridgeArgs($selectedToken, $enteredAmount, commonArgs, selectedNFT, nftIdArray);

      const args = { ...bridgeArgs, tokenIds };

      const txHash = await $bridgeService.bridge(args);

      const explorer = chainConfig[bridgeArgs.srcChainId].urls.explorer;

      infoToast({
        title: $t('bridge.actions.bridge.tx.title'),
        message: $t('bridge.actions.bridge.tx.message', {
          values: {
            token: $selectedToken.symbol,
            url: `${explorer}/tx/${txHash}`,
          },
        }),
      });

      //TODO: everything below should be handled differently for the stepper design. Still tbd
      await pendingTransactions.add(txHash, $network.id);

      successToast({
        title: $t('bridge.actions.bridge.success.title'),
        message: $t('bridge.actions.bridge.success.message'),
      });

      // Let's add it to the user's localStorage
      const bridgeTx = {
        hash: txHash,
        from: $account.address,
        amount: $enteredAmount,
        symbol: $selectedToken.symbol,
        decimals: $selectedToken.decimals,
        srcChainId: BigInt($network.id),
        destChainId: BigInt($destinationChain.id),
        tokenType: $selectedToken.type,
        status: MessageStatus.NEW,
        timestamp: Date.now(),

        // TODO: do we need something else? we can have
        // access to the Transaction object:
        // TransactionLegacy, TransactionEIP2930 and
        // TransactionEIP1559
      } as BridgeTransaction;

      bridgeTxService.addTxByAddress($account.address, bridgeTx);

      nextStep();
    } catch (err) {
      console.error(err);
      handleBridgeError(err as Error);
    }
  }

  $: if ($selectedToken && amountComponent) {
    amountComponent.validateAmount();
  }

  const resetForm = () => {
    //we check if these are still mounted, as the user might have left the page
    if (amountComponent) amountComponent.clearAmount();
    if (recipientComponent) recipientComponent.clearRecipient();
    if (processingFeeComponent) processingFeeComponent.resetProcessingFee();
    if (addressInputComponent) addressInputComponent.clearAddress();

    // Update balance after bridging
    if (amountComponent) amountComponent.updateBalance();
    if (nftIdInputComponent) nftIdInputComponent.clearIds();

    $selectedToken = ETHToken;
    contractAddress = '';
    importMethod === 'scan';
    scanned = false;
    canProceed = false;
    selectedNFT = [];
  };

  /**
   *   NFT Bridge specific
   */
  let activeStep: NFTSteps = NFTSteps.IMPORT;

  const nextStep = () => (activeStep = Math.min(activeStep + 1, NFTSteps.CONFIRM));
  const previousStep = () => (activeStep = Math.max(activeStep - 1, NFTSteps.IMPORT));

  let nftStepTitle: string;
  let nftStepDescription: string;
  let nextStepButtonText: string;

  let contractAddress: Address | '';

  let addressInputComponent: AddressInput;
  let nftIdInputComponent: IdInput;

  let scanning: boolean = false;
  let scanned: boolean = false;

  let selectedNFT: NFT[];
  let canProceed: boolean;
  let foundNFTs: NFT[] = [];

  const scanForNFTs = async () => {
    scanning = true;
    const accountAddress = $account?.address;
    const srcChainId = $network?.id;
    if (!accountAddress || !srcChainId) return;
    const nftsFromAPIs = await fetchNFTs(accountAddress, BigInt(srcChainId));
    foundNFTs = nftsFromAPIs.nfts;
    scanning = false;
    scanned = true;
  };

  const getStepText = () => {
    if (activeStep === NFTSteps.REVIEW) {
      return $t('common.confirm');
    }
    if (activeStep === NFTSteps.CONFIRM) {
      return $t('common.ok');
    } else {
      return $t('common.continue');
    }
  };

  const changeImportMethod = () => {
    importMethod = importMethod === 'manual' ? 'scan' : 'manual';
    resetForm();
  };

  const manualImportAction = () => {
    if (!$network?.id) throw new Error('network not found');
    const srcChainId = $network?.id;
    const tokenId = nftIdArray[0];

    if (contractAddress && srcChainId)
      getTokenWithInfoFromAddress({ contractAddress, srcChainId, owner: $account?.address, tokenId })
        .then((token) => {
          if (!token) throw new Error('no token with info');
          selectedNFT = [token as NFT];
          $selectedToken = token;
        })
        .catch((err) => {
          console.error(err);
          //   detectedTokenType = null;
          //   addressInputState = AddressInputState.Invalid;
        });
    nextStep();
  };

  // Whenever the user switches bridge types, we should reset the forms
  $: $activeBridge && (resetForm(), (activeStep = NFTSteps.IMPORT));

  // Set the content text based on the current step
  $: {
    const stepKey = NFTSteps[activeStep].toLowerCase();
    nftStepTitle = $t(`bridge.title.nft.${stepKey}`);
    nftStepDescription = $t(`bridge.description.nft.${stepKey}`);
    nextStepButtonText = getStepText();
  }
  onDestroy(() => {
    resetForm();
  });
</script>

<div class="f-col">
  <Stepper {activeStep}>
    <Step stepIndex={NFTSteps.IMPORT} currentStepIndex={activeStep} isActive={activeStep === NFTSteps.IMPORT}
      >{$t('bridge.title.nft.import')}</Step>
    <Step stepIndex={NFTSteps.REVIEW} currentStepIndex={activeStep} isActive={activeStep === NFTSteps.REVIEW}
      >{$t('bridge.title.nft.review')}</Step>
    <Step stepIndex={NFTSteps.CONFIRM} currentStepIndex={activeStep} isActive={activeStep === NFTSteps.CONFIRM}
      >{$t('bridge.title.nft.confirm')}</Step>
  </Stepper>

  <Card class="mt-[32px] w-full md:w-[524px]" title={nftStepTitle} text={nftStepDescription}>
    <div class="space-y-[30px]">
      <!-- IMPORT STEP -->
      {#if activeStep === NFTSteps.IMPORT}
        <ImportStep
          bind:importMethod
          bind:canProceed
          bind:selectedNFT
          {scanForNFTs}
          {nftIdArray}
          {contractAddress}
          {foundNFTs}
          bind:scanned
          bind:scanning />
        <!-- REVIEW STEP -->
      {:else if activeStep === NFTSteps.REVIEW}
        <ReviewStep bind:selectedNFT />
        <!-- CONFIRM STEP -->
      {:else if activeStep === NFTSteps.CONFIRM}
        <div class="f-between-center gap-4">
          <ChainSelectorWrapper />
        </div>
      {/if}
      <!-- 
        User Actions
      -->
      {#if activeStep !== NFTSteps.IMPORT}
        <Button
          type="neutral"
          class="px-[28px] py-[14px] rounded-full w-auto flex-1 bg-transparent !border border-primary-brand hover:border-primary-interactive-hover"
          on:click={previousStep}>
          <span class="body-bold">{$t('common.edit')}</span></Button>
      {/if}
      {#if activeStep === NFTSteps.REVIEW}
        <div class="f-between-center w-full gap-4">
          <div class="h-sep" />
          <Actions {approve} {bridge} />
        </div>
      {:else if activeStep === NFTSteps.IMPORT}
        {#if importMethod === 'manual'}
          <div class="f-col w-full">
            <Button
              disabled={!canProceed}
              type="primary"
              class="px-[28px] py-[14px] rounded-full flex-1 w-auto text-white"
              on:click={manualImportAction}><span class="body-bold">{nextStepButtonText}</span></Button>

            <button on:click={() => changeImportMethod()} class="flex justify-center py-3 link">
              {$t('common.back')}
            </button>
          </div>
        {:else if scanned}
          <div class="f-col w-full">
            <Button
              disabled={!canProceed}
              type="primary"
              class="px-[28px] py-[14px] rounded-full flex-1 w-auto text-white"
              on:click={nextStep}><span class="body-bold">{nextStepButtonText}</span></Button>

            <button on:click={resetForm} class="flex justify-center py-3 link">
              {$t('common.back')}
            </button>
          </div>
        {/if}
      {/if}
    </div>
  </Card>
</div>

<OnNetwork change={onNetworkChange} />
<OnAccount change={onAccountChange} />
