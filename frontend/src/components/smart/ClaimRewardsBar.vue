<template>
  <div class="body main-font">
    <b-navbar>
      <b-icon-exclamation-circle-fill class="rewards-claimable-icon" scale="1.2"
      variant="success" :hidden="!canClaimTokens && !canClaimXp" v-tooltip.bottom="'Rewards ready to claim!'"/>

      <b-nav-item disabled><strong>Rewards</strong></b-nav-item>

      <b-nav-item
        class="ml-3"
        :disabled="!canClaimTokens"
        @click="claimSkill(ClaimStage.WaxBridge)"><!-- moved gtag-link below b-nav-item -->
        <span class="gtag-link-others" tagname="claim_skill">
          <strong>SKILL</strong> {{ formattedSkillReward }}
          <strong>Early Withdraw Tax</strong> {{ formattedRewardsClaimTax }}
          <b-icon-question-circle class="centered-icon" scale="0.8" v-tooltip.bottom="'Tax is being reduced by 1% per day'"/>
        </span>
      </b-nav-item>

      <b-nav-item
        class="ml-3"
        :disabled="!canClaimXp"
        @click="onClaimXp">
          <div class="gtag-link-others" v-html="`<strong>XP</strong> ${formattedXpRewards}`"></div>
      </b-nav-item>
    </b-navbar>

    <b-modal class="centered-modal" ref="need-gas-modal" title="Need Gas?"
      @ok="claimSkill(ClaimStage.Stake)" ok-title="Next" @cancel="$router.push({ name: 'portal' })" cancel-title="Go to WAX Bridge" >
        Need Gas? Try our WAX Bridge, which will pay you .5% under market rate to sell your WAX for BNB!
    </b-modal>
    <b-modal class="centered-modal" ref="stake-suggestion-modal" title="Stake Skill"
      @ok="$router.push({ name: 'select-stake-type' })" ok-only ok-title="Go to Stake" >
        If you stake your SKILL now, we will give you a 10% bonus in SKILL that you can use in-game right away!
      <a href="#" @click="claimSkill(ClaimStage.Claim)"> <br>No thanks, I'd rather {{ (this.rewardsClaimTaxAsFactorBN > 0)?"pay " +
        this.formattedTaxAmount + " in taxes and " : ""  }}forfeit my bonus </a>
    </b-modal>
    <b-modal class="centered-modal" ref="claim-confirmation-modal" title="Claim Skill" ok-title="I am sure"
      @ok="onClaimTokens()"> You are about to {{ (this.rewardsClaimTaxAsFactorBN > 0)?"pay " + formattedRewardsClaimTax +
      " tax for early withdrawal, costing you " + this.formattedTaxAmount + " SKILL. You will also " : "" }}
      miss out on {{formattedBonusLost}} bonus SKILL. Are you sure
      you wish to continue? <b>This action cannot be undone.</b>
    </b-modal>
  </div>
</template>

<script lang="ts">
import Vue from 'vue';
import { Accessors } from 'vue/types/options';
import { mapActions, mapGetters, mapState } from 'vuex';
import BN from 'bignumber.js';
import Web3 from 'web3';
import { getCharacterNameFromSeed } from '../../character-name';
import { RequiredXp } from '../../interfaces';
import { ICharacter } from '@/interfaces';

interface StoreMappedState {
  skillRewards: string;
  xpRewards: Record<string, string>;
  ownedCharacterIds: string[];
  directStakeBonusPercent: number;
}

interface StoreMappedGetters {
  ownCharacters: ICharacter[];
  currentCharacter: ICharacter | null;
  maxRewardsClaimTaxAsFactorBN: BN;
  rewardsClaimTaxAsFactorBN: BN;
}

enum ClaimStage {
  WaxBridge = 0,
  Stake = 1,
  Claim = 2
}

interface StoreMappedActions {
  claimTokenRewards(): Promise<void>;
  claimXpRewards(): Promise<void>;
}

export default Vue.extend({
  data() {
    return {
      ClaimStage
    };
  },

  computed: {
    ...(mapState(['skillRewards', 'xpRewards', 'ownedCharacterIds', 'directStakeBonusPercent']) as Accessors<StoreMappedState>),
    ...(mapGetters([
      'ownCharacters', 'currentCharacter', 'maxRewardsClaimTaxAsFactorBN', 'rewardsClaimTaxAsFactorBN'
    ]) as Accessors<StoreMappedGetters>),

    formattedSkillReward(): string {
      const skillRewards = Web3.utils.fromWei(this.skillRewards, 'ether');
      return `${new BN(skillRewards).toFixed(4)}`;
    },

    formattedTaxAmount(): string {
      const skillRewards = Web3.utils.fromWei((parseFloat(this.skillRewards)* parseFloat(String(this.rewardsClaimTaxAsFactorBN))).toString(), 'ether');
      return `${new BN(skillRewards).toFixed(4)}`;
    },

    formattedBonusLost(): string {
      const skillLost = Web3.utils.fromWei((parseFloat(this.skillRewards)*this.directStakeBonusPercent/100).toString(), 'ether');
      return `${new BN(skillLost).toFixed(4)}`;
    },

    formattedRewardsClaimTax(): string {
      const frac =
        this.skillRewards === '0'
          ? this.maxRewardsClaimTaxAsFactorBN
          : this.rewardsClaimTaxAsFactorBN;

      return `${frac.multipliedBy(100).decimalPlaces(0, BN.ROUND_HALF_UP)}%`;
    },

    xpRewardsForOwnedCharacters(): string[] {
      return this.ownedCharacterIds.map(charaId => this.xpRewards[charaId] || '0');
    },

    formattedXpRewards(): string {
      return this.xpRewardsForOwnedCharacters.map((xp, i) => {
        const currentCharacter = this.currentCharacter || { id: null };
        if(!this.ownCharacters[i]) return `${xp}`;
        return  `${this.ownCharacters[i].id === currentCharacter.id ? '<b>' : ''}` +
                `${(this.ownCharacters[i].xp + this.xpRewards[this.ownCharacters[i].id]) as any > RequiredXp(this.ownCharacters[i].level) ? '<u>' : ''}` +
                `${getCharacterNameFromSeed(this.ownCharacters[i].id)} ${xp}` +
                `${(this.ownCharacters[i].xp + this.xpRewards[this.ownCharacters[i].id]) as any > RequiredXp(this.ownCharacters[i].level) ? '</u>' : ''}` +
                `${this.ownCharacters[i].id === currentCharacter.id ? '</b>' : ''}`;
      }).join(', ');
    },

    canClaimTokens(): boolean {
      if(new BN(this.skillRewards).lte(0)) {
        return false;
      }

      return true;
    },

    canClaimXp(): boolean {
      const allXpsAreZeroOrLess = this.xpRewardsForOwnedCharacters.every(xp => new BN(xp).lte(0));
      if(allXpsAreZeroOrLess) {
        return false;
      }

      return true;
    }
  },

  methods: {
    ...(mapActions(['addMoreSkill', 'claimTokenRewards', 'claimXpRewards']) as StoreMappedActions),

    async onClaimTokens() {
      if(this.canClaimTokens) {
        await this.claimTokenRewards();
      }
    },

    async onClaimXp() {
      if(this.canClaimXp) {
        await this.claimXpRewards();
      }
    },

    async claimSkill(stage: ClaimStage) {
      if(stage === ClaimStage.WaxBridge) {
        (this.$refs['need-gas-modal'] as any).show();
      }
      if(stage === ClaimStage.Stake) {
        (this.$refs['stake-suggestion-modal'] as any).show();
      }
      if(stage === ClaimStage.Claim) {
        (this.$refs['stake-suggestion-modal'] as any).hide();
        (this.$refs['claim-confirmation-modal'] as any).show();
      }
    }
  }
});
</script>

<style scoped>

.navbar {
  background: rgb(20,20,20);
  background: linear-gradient(45deg, rgba(20,20,20,1) 0%, rgba(36,39,32,1) 100%);
}

.nav-item {
  margin-top: -24px;
}

.nav-item a {
  padding: 0;
}

.rewards-claimable-icon {
  margin-right: 5px;
  align-self: center;
}
</style>
